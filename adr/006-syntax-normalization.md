# ADR-006: Syntax Normalization — Multiple Source Forms, Single AST

- **Status:** Accepted
- **Date:** 2026-03-19
- **Context:** OpenStrux is designed for LLM generation. Different LLMs
  (and different prompts) will produce syntactically different but
  semantically equivalent code. A language that rejects valid variations
  forces brittle prompt engineering. A language that accepts variations but
  produces different output is non-deterministic. Neither is acceptable.

## Decision

The compiler accepts multiple equivalent syntax forms and normalizes them
to a single canonical AST before IR construction. Syntactically different
but semantically equivalent source files MUST produce identical IR and
identical output.

### Accepted Variations (Panel Shorthand Rules)

| Verbose (always valid) | Shorthand (recommended) |
|----------------------|------------------------|
| `@rod f = filter { arg.predicate = ... }` | `f = filter { predicate: ... }` |
| `cfg.source = db.sql.postgres { ... }` | `source: db.sql.postgres { ... }` |
| `snap db.out.rows -> in.data` | `from: db.rows` (or implicit chain) |
| `@access { intent: { purpose: "x" } }` | `@access { purpose: "x" }` |

All four shorthand rules (drop `@rod`/`=`, drop `cfg.`/`arg.` prefixes,
implicit linear snaps, flatten `@access`) are syntactic sugar that the
compiler resolves deterministically.

### The LLM Contract

An LLM generating OpenStrux code may produce any mix of verbose and
shorthand forms. The contract is:

1. If it parses, it compiles
2. If it compiles, the output is deterministic
3. Two files expressing the same pipeline in different syntax produce
   identical artifacts

The shorthand form is RECOMMENDED for authoring (fewer tokens = fewer
generation errors, lower cost, faster inference). The verbose form is
always valid for backward compatibility and explicit disambiguation.

## Alternatives Considered

**Single canonical syntax only:** Forces LLMs to learn one exact form.
Increases generation failure rate. Requires more prompt engineering to
ensure exact syntax compliance. Rejected.

**Multiple syntaxes with different semantics:** Non-deterministic. Same
pipeline described differently could produce different output. Violates
the determinism guarantee (ADR-000 §4). Rejected.

**Post-processing formatter (like gofmt):** Normalizes source code, not
output. Doesn't solve the problem — the compiler must still handle both
forms. A formatter is useful but orthogonal. Not a substitute.

## Source Location Preservation

The canonical AST **preserves source locations** in the `loc` field of
every IR node (see `SourceSpan` in `@openstrux/ast`). This enables:

- **Diagnostics**: Error messages point to the exact line and column in
  the original source, regardless of which syntax form was used
- **Explain output**: `strux panel build --explain` can show both the
  logical plan (AST-level) and the source location of each rod
- **Debuggability**: A rod written as shorthand `f = filter { ... }` and
  a rod written as verbose `@rod f = filter { arg.predicate = ... }`
  both carry their source position through to the compiled manifest

### What Is NOT Preserved

The **syntax form** itself is not preserved. The IR does not record
whether the source used `@rod` or not, whether `cfg.` prefixes were
present, or whether snaps were implicit or explicit. After
normalization, both forms are identical.

This is an **accepted trade-off**: traceability to source *location*
is preserved; traceability to source *form* is not. A generated
explanation will reference the canonical form (e.g., `cfg.source`,
explicit snaps), which may differ from what the author originally
wrote. This is consistent with how every compiler works — the AST
represents the program, not the formatting.

If round-trip fidelity to the original form is needed (e.g., for
IDE refactoring or formatting), a separate CST (concrete syntax tree)
can be maintained alongside the AST. This is out of scope for v0.5
(see ADR-013).

## Consequences

- The compiler's parser must handle all accepted forms
- The compiled manifest (`mf.strux.json`) always uses the fully qualified
  form (with `cfg.`, `arg.`, explicit snaps) regardless of source syntax
- Source locations (`loc` fields) trace every IR node back to the
  original source file, line, and column
- Testing must verify that verbose and shorthand forms produce identical
  output (conformance fixture requirement)
- The shorthand form is the recommended generation target for LLMs —
  ~142 tokens vs ~336 for self-contained verbose form
- Ambiguity resolution: if a rod type has overlapping cfg/arg names, the
  compiler requires explicit prefixes (no silent disambiguation)
- Syntax form is not preserved — this is an accepted trade-off for
  simplicity. CST preservation is deferred (ADR-013)
