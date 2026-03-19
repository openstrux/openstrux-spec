# ADR-011: Certification Scope Is Not Inherited

- **Status:** Accepted
- **Date:** 2026-03-19
- **Updated:** 2026-03-19
- **Context:** Config inheritance (`strux.context`) allows `@dp`, `@access`,
  `@ops`, and `@sec` to cascade from project to folder to panel. The
  question is whether `@cert` should also inherit — if a project-level
  context declares a certification, do all panels below it automatically
  carry that certification?

## Decision

**No.** Certification (`@cert`) is never inherited at **any** config
level. This applies uniformly across the entire context cascade:

- Project `strux.context` → folder `strux.context`: NOT inherited
- Folder `strux.context` → panel `.strux`: NOT inherited
- Panel → rod: NOT inherited (rod-level `@cert` is independent)
- Nested composition (panel includes another panel): NOT inherited

A `@cert` block in a `strux.context` file is a **compile error**. The
compiler MUST reject it with diagnostic `E_CERT_IN_CONTEXT`. This
eliminates ambiguity — there is no level at which `@cert` can appear
in a context file, and therefore no level at which inheritance is
possible.

### Rationale

- Certification is an **attestation**, not a default. Saying "this panel
  is certified for Postgres" means someone tested THIS panel with
  Postgres. Inheriting it from a parent context would mean "certified
  because the project says so" — which is meaningless.
- Different panels in the same project may be certified for different
  source types (one for Postgres, another for BigQuery). Inheritance
  would either force the lowest common denominator or silently over-claim.
- Certification has a `hash` field — it attests to a specific version of
  the component. Inheritance would break this binding.
- Making `@cert` illegal in context files (not just "ignored") prevents
  misunderstanding. A team cannot accidentally believe their context-level
  cert is propagating.

### What IS Inherited

| Block | Inherited | Why |
|-------|----------|-----|
| `@dp` | Yes | Data protection metadata is organizational (same controller, same DPO) |
| `@access` | Yes (narrowing only) | Access defaults are organizational |
| `@ops` | Yes | Operational defaults are infrastructure-level |
| `@sec` | Yes | Security classification is organizational |
| `@cert` | **No — compile error in context** | Certification is per-component attestation |

## Conformance Cases

### Valid

- `v011-panel-cert.strux` — Panel with its own `@cert` block declaring
  type path scope and content hash. Build succeeds.
- `v011-rod-cert.strux` — Rod within a panel has its own `@cert` block,
  independent of the panel's `@cert`. Build succeeds.
- `v011-no-cert.strux` — Panel without `@cert` in a project that has
  certified panels. Build succeeds (certification is optional, not
  required by default).

### Invalid

- `i011-cert-in-project-context.strux` — Project-level `strux.context`
  contains a `@cert` block. Expected: `E_CERT_IN_CONTEXT`.
- `i011-cert-in-folder-context.strux` — Folder-level `strux.context`
  contains a `@cert` block. Expected: `E_CERT_IN_CONTEXT`.
- `i011-cert-hash-mismatch.strux` — Panel `@cert` block references a
  content hash that does not match the panel's compiled output.
  Expected: `E_CERT_HASH_MISMATCH`.
- `i011-cert-scope-uncovered.strux` — Panel uses type path
  `db.sql.postgres` but `@cert` scope only covers `db.sql.mysql`.
  Expected: `W_CERT_SCOPE_UNCOVERED` (warning — panel works but
  certification does not cover the actual configuration).

## Alternatives Considered

**Inherit cert with override:** Parent context sets a baseline cert, panels
can extend or narrow. Problem: creates false confidence. A panel might
appear certified because it inherited a scope it was never tested with.
Rejected.

**Ignore cert in context (silent):** Allow `@cert` in context files but
silently ignore it. Problem: teams would write it, believe it works, and
discover the gap only at audit time. Rejected in favor of compile error.

## Consequences

- Every panel that needs certification must declare its own `@cert` block
- Every rod that needs independent certification must declare its own
  `@cert` block
- `@cert` in any `strux.context` file is `E_CERT_IN_CONTEXT`
- The `@cert` block references specific type paths and content hashes
- This is more verbose but prevents certification fraud
- Tooling can assist by auto-generating `@cert` blocks after test runs
- The `policy_verification` field (ADR-010) is owned by audit metadata
  and may be mirrored into `@cert` for source-level visibility. It is
  per-component, consistent with this decision
