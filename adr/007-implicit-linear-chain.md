# ADR-007: Implicit Linear Chain as Default Rod Wiring

- **Status:** Accepted
- **Date:** 2026-03-19
- **Context:** Most panels are linear: read → filter → transform → write.
  Requiring explicit `snap` declarations for every rod-to-rod connection
  adds ~2 tokens per connection with zero information — the linear case is
  predictable from declaration order.

## Decision

Rods in declaration order form an implicit chain: each rod reads from the
**default output** of the immediately preceding rod into its own **default
input**. Explicit wiring (`from:` or `snap`) is only required when the
chain branches or when reading from a non-default output.

### Formal Rules

1. **First rod** has no implicit input (it is a source: read-data, receive)
2. **Subsequent rods** without `from:` read from the default output of the
   immediately preceding rod
3. **`from: rod.knot`** overrides the implicit chain
4. **`from: [rod1, rod2]`** for multi-input rods (first = left, second = right)
5. **Multiple outputs** (filter.match + filter.reject): implicit chain
   follows the default (match). Non-default outputs require explicit `from:`
6. **No implicit chain across branches** — after a rod with multiple
   downstreams, the next rod follows the default output; others must be
   wired explicitly
7. **Declaration order, not topological order** — the author controls the
   chain by ordering declarations; the compiler validates acyclicity

### Default Knots Per Rod

Each rod type defines which output is the default (implicit chain target)
and which input is the default (implicit chain receiver). See
`panel-shorthand.md` for the full table.

## Alternatives Considered

**Always explicit snaps:** Safe but verbose. Adds ~67 tokens (32%) of
structural noise to a typical panel. The linear case is so common that
requiring explicit wiring is pure ceremony. Rejected as the default
(but remains valid syntax).

**Topological inference (auto-wiring by type compatibility):** Too magical.
The author loses control over execution order. Two rods with compatible
types would auto-connect even if unrelated. Rejected.

**Named channels / ports:** More flexible than implicit chain but more
complex. Overkill for the common linear case. The `from:` override
provides equivalent power when needed. Rejected as the default mechanism.

## Consequences

- Panel authoring is concise: a 5-rod linear pipeline needs zero `snap`
  or `from:` declarations
- Declaration order matters — reordering rods changes the implicit chain
- LLMs generate correct panels by simply listing rods in execution order
- Non-linear topologies (fan-out, merge, DLQ routing) always require
  explicit `from:` — this is a feature, making branches visible
- The compiled manifest always contains explicit snaps regardless of
  source syntax (normalization per ADR-006)
