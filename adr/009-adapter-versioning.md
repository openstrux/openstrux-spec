# ADR-009: Adapter Versioning Independent of Rod Versions

- **Status:** Accepted
- **Date:** 2026-03-19
- **Updated:** 2026-03-19
- **Context:** Adapters translate rod config into target-specific code
  (e.g., PostgresConfig → ReadFromJdbc). When Postgres 17 ships new
  features, the adapter needs updating. Should this force a new rod
  version?

## Decision

Adapter versions are **independent** of rod versions. A rod at version
1.0 can work with adapter versions 1.0, 1.1, 2.0 as long as the adapter
satisfies the interface contract defined by the union leaf type.

### Rationale

- Rod cores are pure logic — they don't change when a database releases
  a new version
- Adapter updates are driven by external systems (database releases,
  framework updates), not by OpenStrux language changes
- Independent versioning allows hotfixing an adapter without touching
  tested and certified rod logic
- The lock file (`snap.lock`) pins both rod and adapter versions
  independently for reproducibility

## 1. Interface Contract

Adapter compatibility is defined against a **versioned union-leaf
interface artifact**, not a type path string. The interface artifact is
a published hub object that contains:

- The union leaf type definition (e.g., `PostgresConfig@1.2`)
- The field set, types, and constraints the adapter must consume
- A content hash binding the interface to a specific version

Adapters declare which interface version(s) they implement. Semver
rules apply to the interface artifact:

| Interface change | Version bump | Adapter impact |
|---|---|---|
| New optional field | Minor | Compatible — adapter ignores unknown fields |
| Field type change | Major | Breaking — adapter must update |
| Field removed | Major | Breaking — adapter may depend on it |
| New required field | Major | Breaking — adapter must supply it |

A rod manifest declares `adapter-compat` as a semver range against the
interface artifact version:

```
adapter-compat: "PostgresConfig@^1.0"
```

This means: "any adapter that implements PostgresConfig interface 1.x".

## 2. Panel Source Cannot Pin Adapter Versions

Panel `.strux` source MUST NOT contain adapter version references.
The source declares **what** (type path + config), not **which adapter
version**. This keeps source files declarative and target-independent.

However, **resolver policy** may constrain adapter selection before lock
generation. Resolver policy is configured in `strux.context` or project
settings:

```
@context {
  @resolve {
    adapter-policy: "certified-only"         // only certified adapters
    adapter-channel: "stable"                // stable channel, not canary
    adapter-constraint: "PostgresConfig -> beam-python@>=2.1"  // explicit floor
  }
}
```

The resolver evaluates policy, selects the best matching adapter, and
writes the **exact version + hash** into `snap.lock`. After lock
generation, the lock is authoritative — resolver policy is not
re-evaluated.

## 3. Hub Publication Rules

The hub MUST enforce at publication time:

- **Rod publication:** If a rod manifest declares `adapter-compat:
  "PostgresConfig@^1.0"` for target `beam-python`, the hub checks
  whether at least one certified adapter exists that satisfies the
  range for that target. If none exists, the rod is published with
  status `non-resolvable` and a warning. Builds referencing a
  non-resolvable rod MUST fail with diagnostic `E_NO_ADAPTER`.

- **Adapter publication:** The adapter declares which interface
  version(s) it implements and which targets it supports. The hub
  validates that the adapter's declared interface matches the
  published interface artifact schema (structural compatibility check).

- **Interface publication:** A new major version of an interface
  artifact triggers a hub-wide compatibility scan. Adapters and rods
  referencing the previous major are flagged for review.

## 4. Lock File Requirements

`snap.lock` MUST record for each adapter dependency:

```json
{
  "adapter": "PostgresConfig -> beam-python",
  "version": "2.1.3",
  "hash": "sha256:a1b2c3...",
  "interface": "PostgresConfig@1.2",
  "interface_hash": "sha256:d4e5f6...",
  "cert": {
    "level": "tested",
    "scope": { "source": "db.sql.postgres" },
    "expires": "2027-01-15"
  }
}
```

The lock pins: exact adapter version, adapter content hash, interface
version, interface content hash, and certification snapshot at lock time.

## 5. Conformance Cases

### Valid

- `v009-adapter-lock-complete.strux` — Panel with fully resolved adapter
  lock entry; build succeeds.
- `v009-adapter-minor-upgrade.strux` — Lock references adapter 2.1.3,
  hub has 2.2.0 available; lock is authoritative, build uses 2.1.3.

### Invalid

- `i009-missing-adapter-lock.strux` — Panel references a type path with
  no adapter entry in `snap.lock`. Expected: `E_MISSING_ADAPTER_LOCK`.
- `i009-incompatible-major.strux` — Lock references adapter implementing
  `PostgresConfig@1.x` but rod requires `PostgresConfig@^2.0`. Expected:
  `E_ADAPTER_INCOMPATIBLE`.
- `i009-multiple-adapters.strux` — Resolver finds two adapters matching
  the same interface + target with no disambiguation policy. Expected:
  `E_AMBIGUOUS_ADAPTER`.
- `i009-cert-scope-mismatch.strux` — Locked adapter's certification
  scope does not cover the panel's type path. Expected:
  `W_ADAPTER_CERT_SCOPE` in development builds; promoted to
  `E_ADAPTER_CERT_SCOPE` in `--locked` and audit modes (see below).

## 6. Certification Scope Enforcement by Build Mode

The manifesto requires that builds or audits MUST fail when a system
uses configuration outside declared certification scope. To balance
development agility with production safety, enforcement varies by mode:

| Diagnostic | `strux build` (dev) | `strux build --locked` (CI/audit) |
|---|---|---|
| `W_ADAPTER_CERT_SCOPE` | Warning — adapter works but cert doesn't cover the type path | Promoted to `E_ADAPTER_CERT_SCOPE` — **build fails** |
| `W_CERT_EXPIRED` | Warning — cert has expired | Promoted to `E_CERT_EXPIRED` — **build fails** |
| `W_UNCERTIFIED` | Warning — adapter has no certification at all | Promoted to `E_UNCERTIFIED` — **build fails** |

In development, warnings let engineers iterate without blocking on
certification. In CI and audit (`--locked`), the manifesto's fail-closed
requirement is enforced: no uncertified or out-of-scope configuration
reaches production.

This distinction MUST be documented in `strux build --help` and in the
conformance suite. The conformance cases above test both modes.

## Alternatives Considered

**Coupled versioning (rod + adapter = one artifact):** Simpler mental
model but forces unnecessary churn. A Postgres adapter bugfix would
require re-certifying the rod core. Rejected.

**Panel-level adapter pinning:** Panels declare `cfg.adapter-version =
"2.1.3"`. Mixes deployment concerns with business logic. Makes source
files target-dependent. Rejected — resolver policy handles this.

## Consequences

- Hub must support independent version resolution for rods and adapters
- Hub must enforce publication-time resolvability checks
- Lock file must track adapter versions, hashes, and cert snapshots
  separately from rod versions
- Certification can target adapter versions independently
- Resolver policy is a new concept in `strux.context` (see ADR-012)
- Builds with non-resolvable rods fail explicitly, not silently
