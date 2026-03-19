# OpenStrux v0.4 — Conformance Specification

## Conformance Classes

A conformant implementation falls into one or more of the following
classes:

| Class | Responsibility | MUST support |
|-------|---------------|-------------|
| **Parser** | `.strux` source → AST | All grammar productions in [grammar.md](grammar.md), both verbose and shorthand forms |
| **Validator** | AST → validated IR | Type checking, scope verification, snap graph validation, pushdown compatibility |
| **Emitter** | IR → target code | At least one emission target, deterministic output |

<!-- GAP v0.6.0.alpha: Add a **Manifest** conformance class covering
     compiled manifest output (mf.strux.json). Per ADR-013, the manifest
     MUST include structured explanation metadata under a schema-defined
     audit field. This class should verify audit field presence, source
     location traceability, and pushdown/policy summary correctness. -->

An implementation MAY claim conformance to one or more classes
independently. A full toolchain (parser + validator + emitter) claims
**full conformance**.

## Normative vs Informative Sections

The following spec files are **normative** — conformant implementations
MUST comply:

- [grammar.md](grammar.md) — syntax and parsing rules
- [type-system.md](type-system.md) — type forms, paths, narrowing
- [ir.md](ir.md) — IR structure and invariants
- [semantics.md](semantics.md) — evaluation behavior
- [locks.md](locks.md) — lock creation, consumption, determinism
- [conformance.md](conformance.md) — this document
- [access-context.strux](access-context.strux) — AccessContext types
- [expressions.strux](expressions.strux) — expression AST types
- [expression-shorthand.md](expression-shorthand.md) — shorthand grammar
  and pushdown rules

The following are **informative** — they provide rationale and context
but do not define conformance requirements:

- [overview.md](overview.md) — language overview
- [design-notes.md](design-notes.md) — design rationale
- [panel-shorthand.md](panel-shorthand.md) — shorthand rules (normative
  content is incorporated into [grammar.md](grammar.md))
- [config-inheritance.md](config-inheritance.md) — context cascade
  (normative content is incorporated into [semantics.md](semantics.md))

## Conformance Fixtures

Conformance fixtures are `.strux` files in the `conformance/` directory
used to test implementation correctness.

### Valid Fixtures (`conformance/valid/`)

A valid fixture MUST parse and typecheck clean on any conformant parser
and validator. Each file contains a comment header:

```
// fixture: v001-record-basic
// expect: valid
// covers: record declaration, primitive field types
```

A conformant parser MUST accept all valid fixtures without error.
A conformant validator MUST typecheck all valid fixtures without
diagnostics.

### Invalid Fixtures (`conformance/invalid/`)

An invalid fixture MUST fail with a specific diagnostic code on any
conformant validator. Each file contains:

```
// fixture: i001-unknown-type
// expect: invalid
// error-code: E_UNKNOWN_TYPE
// covers: type resolution failure
```

A conformant validator MUST reject all invalid fixtures and MUST produce
a diagnostic whose code matches the `error-code` header.

### Golden Fixtures (`conformance/golden/`)

A golden fixture pairs a `.strux` source file with an expected output
file for a specific emission target. A conformant emitter MUST produce
output matching the golden file byte-for-byte (after normalizing
platform-specific line endings).

<!-- GAP: Golden fixture format and naming convention not yet defined.
     Will be specified when the first emission target (Beam Python)
     reaches implementation. -->

<!-- GAP v0.6.0.alpha: Add golden fixtures for compiled manifests
     (conformance/golden/*.mf.strux.json). These MUST verify that the
     audit/explanation metadata required by ADR-013 is present and
     structurally correct. Also add valid fixtures for panels that
     produce explanation output, and invalid fixtures for manifests
     missing required audit fields. -->

## Diagnostic Format

Diagnostics produced by a conformant implementation MUST include:

| Field | Required | Description |
|-------|----------|-------------|
| `code` | YES | Machine-readable code (e.g., `E_UNKNOWN_TYPE`) |
| `severity` | YES | `error` or `warning` |
| `message` | YES | Human-readable description |
| `file` | YES | Source file path |
| `line` | YES | Line number (1-based) |
| `column` | MAY | Column number (1-based) |

### Diagnostic Code Prefixes

| Prefix | Category |
|--------|----------|
| `E_` | Error — compilation MUST fail |
| `W_` | Warning — compilation MAY proceed |

### Defined Diagnostic Codes

| Code | Meaning |
|------|---------|
| `E_UNKNOWN_TYPE` | Reference to a type name that is not declared |
| `E_MISSING_FIELD` | Required field omitted in a record value |
| `E_TYPE_MISMATCH` | Value does not match the expected type |
| `E_AMBIGUOUS_KNOT` | cfg/arg name collision — explicit prefix required |
| `E_UNRESOLVED_PATH` | Type path does not resolve to a concrete type |
| `E_PUSHDOWN_MISMATCH` | Source-specific expression incompatible with upstream source |
| `E_SCOPE_WIDENING` | Child `@access` scope wider than parent |
| `E_CYCLE` | Snap graph contains a cycle |
| `E_UNRESOLVED_SOURCE` | Named source reference (`@name`) not found in context |
| `E_MISSING_DP` | Panel has no `@dp` metadata (after context merge) |
| `W_UNCONNECTED_ERR` | Rod `err` knot not wired and no `@ops` fallback |

<!-- GAP: This list will grow as more error conditions are specified. -->

## Conformance Levels

### Level 1: Minimal (Parser)

A Level 1 implementation:

- MUST parse all grammar productions (verbose and shorthand)
- MUST produce an AST matching the structure in [ir.md](ir.md)
- MUST accept all valid conformance fixtures
- MUST reject syntactically invalid input with a diagnostic

### Level 2: Validated (Parser + Validator)

A Level 2 implementation additionally:

- MUST resolve all type references and type paths
- MUST verify snap graph acyclicity
- MUST verify AccessContext narrowing
- MUST verify pushdown compatibility
- MUST reject all invalid conformance fixtures with correct diagnostic
  codes

### Level 3: Full (Parser + Validator + Emitter)

A Level 3 implementation additionally:

- MUST emit target-specific code for at least one target
- MUST produce deterministic output (same source + same lock = same
  output)
- MUST match golden fixtures for supported targets
- MUST consume `snap.lock` for reproducible builds

<!-- GAP v0.6.0.alpha: Level 3 should additionally require:
     - MUST produce a compiled manifest (mf.strux.json) with structured
       explanation/audit metadata (ADR-013)
     - MUST include source locations (loc fields, per ADR-006) in the
       audit metadata for every IR node
     - Golden manifest fixtures MUST match for supported targets -->

## How to Report Non-Conformance

When a conformance fixture produces an unexpected result, the
implementation SHOULD report:

1. Fixture ID (e.g., `v001-record-basic`)
2. Expected result (`valid` or `invalid` with error code)
3. Actual result (success, or diagnostic code produced)
4. Implementation version and target

## Scope

This section is normative. All conformance requirements use RFC 2119
keywords. The fixture set in `conformance/` is the authoritative test
suite for conformance claims.
