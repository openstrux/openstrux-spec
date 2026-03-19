# ADR-013: Reverse Translation Scope

- **Status:** Accepted
- **Date:** 2026-03-19
- **Updated:** 2026-03-19
- **Context:** The manifesto principle P4 (Human-Translatable) SHOULD-2
  states "human-readable outputs SHOULD preserve traceability back to
  source constructs." ADR-006 establishes that the compiler normalizes
  multiple source forms into a single canonical AST and preserves source
  locations but not syntax form. The question is whether OpenStrux
  should support **reverse translation**: converting generated code
  (Beam Python, TypeScript) or compiled manifests back into `.strux`
  source.

## Decision

**Option A+D: Defer reverse translation. Address P4 SHOULD-2 with
explanation generation from the IR.**

Forward compilation is the critical path for v0.5. Explanation
generation directly addresses the manifesto's traceability requirement
using existing infrastructure (`loc` fields, `--explain` output).
Full reverse translation (manifest → .strux, code → .strux) is
deferred.

## Problem Space

Reverse translation has three potential use cases:

### 1. Code → .strux Compression

An organization has existing Beam Python or TypeScript pipelines and
wants to express them in OpenStrux to gain compliance metadata, type
safety, and certification. This requires parsing target-language code
and reconstructing the `.strux` source.

**Difficulty:** Very high. Target code may use patterns that have no
direct OpenStrux equivalent. Adapter-specific logic (connection
pooling, retry, error handling) lives below the abstraction level
OpenStrux operates at.

**Status:** Deferred indefinitely. May never be needed if adoption is
primarily greenfield.

### 2. Manifest → .strux Decompilation

Given a compiled `mf.strux.json`, reconstruct a valid `.strux` source
that would produce the same manifest.

**Difficulty:** Medium. The manifest contains fully resolved config,
expression ASTs, and snap graphs. Reversing it to source is
mechanically possible but lossy — context inheritance, shorthand
forms, and named source references cannot be recovered (they were
flattened during compilation).

**Status:** Deferred to v0.6 when the manifest format is stable. When
implemented, output MUST be verbose form (self-contained, no context
refs, no shorthand) to guarantee correctness. Shorthand recovery is
an optional optimization.

### 3. Explanation Generation (Accepted for v0.5)

Generate human-readable descriptions of what a panel does, tracing
back to source constructs. This is the P4 SHOULD-2 requirement.

**Difficulty:** Low with source locations. ADR-006 preserves `loc`
fields on every IR node, so explanations can reference exact source
positions even though the AST is in canonical form.

## Explanation Template Format

`strux panel build --explain` produces both the physical plan
(ADR-015) and a human-readable explanation:

```
Panel 'user-analytics' (user-analytics.strux:1):
  1. Reads from PostgreSQL (user-analytics.strux:4)
     - Source: db.sql.postgres, database: "production"
     - Access: purpose=geo_segmentation, basis=legitimate_interest, op=read
  2. Filters rows (user-analytics.strux:5)
     - Predicate: address.country IN ("ES", "FR", "DE") AND deleted_at IS NULL
     - Pushed to source: yes (fused with step 1)
  3. Pseudonymizes PII fields (user-analytics.strux:6)
     - Fields: full_name, email, national_id
     - Algorithm: sha256
  4. Writes to BigQuery (user-analytics.strux:7)
     - Target: db.sql.bigquery, dataset: "eu_users"
  5. Dead-letter queue (user-analytics.strux:8)
     - Target: stream.pubsub, topic: "dlq"
     - Receives: rejected rows from step 2

Pushdown: 1 of 4 operations pushed (25%)
Escape hatches: 0 of 4 expressions (0%)
Policy verification: all static
```

### Explanation Properties

- Every step references a source location (`file:line`)
- The explanation uses the canonical form (cfg/arg names, explicit
  snap wiring), not the original syntax form — consistent with
  ADR-006's accepted trade-off
- Access context, pushdown status, and policy verification are
  included for audit traceability
- The explanation is generated from the IR, not from source — it
  reflects what the compiler understood, not what was written

### Machine-Readable Explanation

The compiled manifest (`mf.strux.json`) MUST include structured
explanation metadata under a schema-defined audit field. The exact
field name and structure are defined by the manifest schema (see
`packages/manifest/` in openstrux-core). This enables automated
compliance tooling to consume the same traceability information
programmatically.

## CST (Concrete Syntax Tree) — Deferred

ADR-006 references the possibility of maintaining a CST alongside the
AST for full round-trip fidelity. This would enable:

- Formatter that preserves author style
- IDE refactoring that doesn't rewrite the entire file
- Decompilation that can choose shorthand vs verbose output

This is explicitly **out of scope for v0.5**. The AST with `loc`
fields is sufficient for all v0.5 requirements. CST can be added as
a non-breaking enhancement when IDE tooling demands it.

## Alternatives Considered

**Defer all (no explanation generation):** P4 SHOULD-2 remains
unaddressed. The `--explain` output would show only the physical plan
(ADR-015) without human-readable traceability. The explanation format
is low-cost to implement and high-value for adoption. Rejected.

**Full round-trip with CST (Option C):** Maximum fidelity but
significant implementation complexity. CST maintenance is a parser
burden. Not required for any MUST condition in v0.5. Deferred.

**Manifest → .strux only (Option B):** Useful for audit but doesn't
address P4 SHOULD-2 (which asks for human-readable explanations, not
reconstructed source). Deferred to v0.6 as a complementary feature.

## Consequences

- P4 SHOULD-2 is addressed by explanation generation from the IR
- `strux panel build --explain` is the primary traceability tool,
  producing both physical plan (ADR-015) and human-readable explanation
- `mf.strux.json` MUST include structured explanation metadata under a
  schema-defined audit field (exact name owned by the manifest schema)
- Source locations (`loc` fields) trace every IR node back to the
  original source file, line, and column (ADR-006)
- Syntax form is not preserved — accepted trade-off (ADR-006)
- CST is explicitly out of scope for v0.5
- Manifest → .strux decompilation is deferred to v0.6
- Code → .strux migration is deferred indefinitely
