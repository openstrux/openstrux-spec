# ADR-010: Policy Store Mechanism (Tiered)

- **Status:** Accepted
- **Date:** 2026-03-19
- **Updated:** 2026-03-19
- **Context:** The `guard` rod references policies by name:
  `policy("data-engineering-read")`. The `@access` block references scoped
  policies: `scope: policy("default-read")`. These policy names must
  resolve to actual policy definitions. Where do those definitions live?

## Decision

**Tiered policy model.** Three tiers, each with a different verification
guarantee. Organizations use the tier that matches their policy
complexity and change velocity.

| Tier | Where defined | Scope narrowing verified? | Lock pinned? |
|---|---|---|---|
| Inline | Guard rod shorthand | Yes — full static analysis | N/A (source is the definition) |
| Hub policy | Named artifact in hub | Yes — compiler loads and evaluates statically | Yes — version + hash |
| External | Engine reference (OPA, Cedar, custom) | No — opaque at compile time | Partial — engine ref pinned, not policy content |

### Rationale

This matches how organizations actually manage policies:

- **Simple rules inline** — `principal.roles HAS "admin"` in a guard rod.
  No external dependency. Compiler verifies scope narrowing fully.
- **Shared rules in a registry** — `policy("data-engineering-read")` as a
  versioned hub artifact. Reusable across panels. Compiler loads the
  artifact and evaluates it like inline predicates.
- **Complex rules in a dedicated engine** — `opa: data.authz.allow` for
  organizations with existing policy infrastructure. Runtime-evaluated.
  Compiler cannot prove scope narrowing.

The key insight: **all three tiers produce the same runtime behavior**
(allow/deny). They differ only in where the policy is defined and
whether the compiler can statically verify it.

## 1. Inline Policies

Guard rod shorthand predicates (see
[expression-shorthand.md §8](../specs/core/expression-shorthand.md)):

```
guard = guard {
  policy: principal.roles HAS "analyst" AND element.country IN ("ES", "FR")
}
```

- Defined in panel source — no external reference
- Compiler parses, type-checks, and verifies scope narrowing at build time
- Compiled to expression AST in the IR (see `PortablePolicy` in
  [ir.md](../specs/core/ir.md))
- No lock entry needed — the source is the definition

## 2. Hub Policies

Named, versioned artifacts published to the hub. Format is a `.strux`
file containing guard predicates:

```
@policy data-engineering-read {
  version: "1.2.0"

  // The predicate — same syntax as inline guard policy
  rule: principal.roles HAS ANY ("data-engineer", "analyst")
        AND intent.operation == "read"
        AND intent.basis IN ("legitimate_interest", "contract")

  // Metadata
  owner: "platform-team"
  description: "Read access for data engineering roles"
}
```

### Hub Policy Resolution

1. Panel references `scope: policy("data-engineering-read")`
2. Compiler resolves name to hub artifact via resolver (same as adapter
   resolution in ADR-009)
3. Compiler loads the policy `.strux`, parses it, and inlines the
   predicate into the IR
4. Scope narrowing is verified statically — the compiler treats the
   resolved predicate identically to inline
5. `snap.lock` pins the policy version and hash:

```json
{
  "policy": "data-engineering-read",
  "version": "1.2.0",
  "hash": "sha256:f1a2b3...",
  "resolved_from": "hub:openstrux/policies"
}
```

### Hub Policy Publication

Hub policies follow the same publication model as rods and adapters
(see ADR-009, ADR-012):

- Versioned with semver
- Content-hashed for integrity
- Certifiable independently
- Lock-pinned for reproducibility

## 3. External Policies

References to external policy engines. The compiler treats these as
opaque — it cannot evaluate the predicate or verify scope narrowing.

```
guard = guard { policy: opa: data.authz.allow }
guard = guard { policy: cedar: Action::"read", Resource::"users" }
```

### Verification Status

`policy_verification` is an **audit metadata field** that records how
a policy was verified. Its authoritative home is the compiled manifest's
audit section (`mf.strux.json → audit.policy_verification`). It MAY
be mirrored into the `@cert` block for source-level visibility:

```
@rod guard = guard {
  policy: opa: data.authz.allow
  @cert {
    policy_verification: "runtime"    // mirror of audit metadata
  }
}
```

The compiler always writes `policy_verification` into audit metadata
in the manifest. If the author also declares it in `@cert`, the
compiler validates consistency — a mismatch is `E_CERT_AUDIT_CONFLICT`.

`policy_verification` values:

| Value | Meaning | Set by |
|---|---|---|
| `"static"` | Compiler verified scope narrowing | Compiler (default for inline and hub) |
| `"runtime"` | Policy evaluated at runtime only; scope narrowing unverified | Compiler (auto for external refs) |
| `"external-audited"` | Runtime-evaluated but independently audited (e.g., OPA test suite) | Author (explicit declaration) |

**Ownership:** Audit metadata owns the field. The compiler computes it
automatically based on policy tier. The `@cert` block may mirror it for
source-level readability, but the manifest's audit section is
authoritative. This separation keeps certification scope (what type
paths a component is tested with) cleanly distinct from policy
provenance (how authorization rules are evaluated).

### Compiler Behavior for External Policies

- Compiler emits `W_POLICY_OPAQUE` warning for any external policy ref
  without `policy_verification`
- If `policy_verification: "runtime"` is present, the warning is
  suppressed — the trade-off is acknowledged
- Scope narrowing analysis marks the guard rod's output scope as
  `unverifiable` — downstream rods that depend on narrowed scope
  receive a `W_SCOPE_UNVERIFIED` warning
- Builds do NOT fail on external policies — this is a warning, not an
  error. Organizations choose their verification tier.

### Lock Entry for External Policies

```json
{
  "policy": "opa:data.authz.allow",
  "type": "external",
  "engine": "opa",
  "endpoint": "https://opa.internal/v1/data/authz/allow",
  "policy_verification": "runtime",
  "pinned_at": "2026-03-19T10:00:00Z"
}
```

The lock pins the engine reference and endpoint at lock time. It does
NOT pin policy content (that's the point of external policies — they
change without recompilation). The `pinned_at` timestamp supports audit.

## 4. Scope Narrowing Across Tiers

| Tier | Compiler can narrow? | Downstream effect |
|---|---|---|
| Inline | Yes | Full static narrowing; downstream rods see narrowed scope |
| Hub | Yes | Same as inline after resolution |
| External | No | Downstream scope is `unverifiable`; `W_SCOPE_UNVERIFIED` emitted |

When a panel mixes tiers (e.g., hub policy on `@access` + external
policy on a guard rod), the **most restrictive verification status
wins** for downstream rods. If any policy in the chain is unverifiable,
downstream scope narrowing is unverifiable.

## 5. Conformance Cases

### Valid

- `v010-inline-policy.strux` — Guard rod with inline predicate; scope
  narrowing verified statically. Build succeeds with no warnings.
- `v010-hub-policy.strux` — Panel references `policy("data-engineering-read")`;
  compiler resolves from hub, verifies scope narrowing. Build succeeds.
- `v010-external-policy-ack.strux` — Guard rod with `opa:` reference and
  `policy_verification: "runtime"`. Build succeeds with no warnings
  (trade-off acknowledged).

### Invalid / Warning

- `i010-external-no-ack.strux` — Guard rod with `opa:` reference but no
  `policy_verification` field. Expected: `W_POLICY_OPAQUE`.
- `i010-scope-unverified-downstream.strux` — Guard rod with external
  policy followed by a rod that depends on narrowed scope. Expected:
  `W_SCOPE_UNVERIFIED` on the downstream rod.
- `i010-hub-policy-not-found.strux` — Panel references
  `policy("nonexistent")`. Expected: `E_POLICY_NOT_FOUND`.

## Alternatives Considered

**Option A: Hub artifact only.** Fits the existing model but too rigid
for organizations with fast-changing authorization rules. Policy changes
would require artifact publication, lock update, and redeployment.
Rejected as sole mechanism — retained as tier 2.

**Option B: External engine only.** Leverages mature engines but
eliminates compile-time verification entirely. The compiler becomes
unable to prove any scope narrowing. Rejected as sole mechanism —
retained as tier 3.

**`scope: "policy-unverified"` annotation.** Overloads the certification
scope field with policy provenance information. Conflates "what type
paths is this tested with" (cert scope) and "how was this policy
verified" (verification status). Rejected in favor of a dedicated
`policy_verification` field.

## Consequences

- Hub must support policy artifacts as a first-class artifact type
  alongside rods and adapters
- Policy artifacts use the same publication, versioning, and lock
  model as adapters (ADR-009)
- `policy_verification` is a new field in `@cert` / audit metadata
- Compiler must implement `W_POLICY_OPAQUE` and `W_SCOPE_UNVERIFIED`
  diagnostics
- Organizations can migrate incrementally: start with inline, extract
  to hub when shared, delegate to external when policy complexity
  demands it
- Mixed-tier panels are explicitly supported with clear verification
  status propagation
