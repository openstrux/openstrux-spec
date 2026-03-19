# v0.4 — Type System Refinement

## New Concepts

| Concept | File | What it is |
|---------|------|------------|
| Union types | `00-strux-type-system.md` | Discriminated unions — third strux form alongside record and enum |
| Type paths | `00-strux-type-system.md` | Dot-separated paths through union trees (`db.sql.postgres`) |
| Adapters | `00-strux-type-system.md` | Per-leaf, per-target I/O implementations — separate from rod logic |
| AccessContext | `01-access-context.strux` | Implicit context: Principal (who) + Intent (why) + Scope (what) |
| DataSource hierarchy | `02-datasource-hierarchy.strux` | Full union tree: stream/db × sql/nosql × concrete providers |
| read-data rod | `03-read-data-rod.strux` | Universal source rod replacing read-db — handles all datasources |
| 18 basic rods | `05-basic-rods.md` | Full rod taxonomy: I/O-Data(2), I/O-Service(3), Computation(7), Control(2), Compliance(3), Topology(1) |
| Expression type system | `06-expressions.strux` | Typed expression AST: FilterExpr, ProjectionExpr, AggregationExpr — with source-specific variants and pushdown rules |
| Expression shorthand | `08-expression-shorthand.md` | SQL-like compact source form that compiles to full strux AST. 8 categories: filter, projection, aggregation, group key, join, sort, split routes, guard policy |
| Compact syntax reference | `09-syntax-reference.md` | ~2,000 token reference for LLM system prompts — all rod signatures, expression grammar, type paths |
| Config inheritance | `10-config-inheritance.md` | `strux.context` files: project → folder → panel cascade for @dp, @access, named @source/@target. Panels carry only deltas |
| Panel shorthand | `11-panel-shorthand.md` | Implicit linear snaps, drop `@rod`/`cfg.`/`arg.` prefixes, flatten @access. 9 lines / ~142 tokens (16% more compact than v0.3 with far richer semantics) |

## Breaking Changes from v0.3

| v0.3 | v0.4 | Migration |
|------|------|-----------|
| `@strux X = enum { a, b }` | Unchanged | Enums still work as-is |
| `@strux X { fields }` | Unchanged | Records still work as-is |
| No union types | `@strux X = union { tag: Type }` | New syntax, additive |
| `cfg.db string ["postgres"]` | `cfg.source DataSource` | Migrate to union-typed cfg |
| `arg.host`, `arg.port` (loose) | Part of `PostgresConfig` (in union) | Config moves inside union variant |
| `read-db` rod | `read-data` rod | Broader scope, same pattern |
| No access control | `AccessContext` implicit knot | New, panels add `@access` block |
| No adapter concept | `@adapter Type -> target { ... }` | New, hub artifact |
| String predicates | Expression shorthand (SQL-like) | Richer syntax but same token cost |

## Backward Compatibility

All v0.3 syntax remains valid. Union types and AccessContext are additive.
The migration from `read-db` to `read-data` is recommended but not required —
`read-db` can continue to exist as a narrower rod that only handles databases.

## Manifesto Review Status

Reviewed against `MANIFESTO_OBJECTIVES.md` in `07-manifesto-review.md`.

| Principle | Status | Notes |
|-----------|--------|-------|
| AI-native | WARN | MUST #4,#5 need benchmark suite |
| Token-efficient | PASS (conditional) | MUST #1 needs benchmark. All SHOULD pass with config inheritance |
| Certified by design | PASS | Strongest area |
| Human-translatable | PASS | Shorthand IS the human form; AST is the machine form |
| Built for performance | PASS | Pushdown is the key feature |
| Structure first | PASS | |
| Trust built in | PASS | |

Resolved actions: expression shorthand, syntax reference.
Remaining: benchmark suite, multi-target compression, @cert update, policy store.

## Files

```
spec/v0.4-types/
├── 00-strux-type-system.md       # Union types, type paths, adapters, grammar
├── 01-access-context.strux       # Principal, Intent, Scope, AuthzResult
├── 02-datasource-hierarchy.strux # DataSource union tree (stream + db)
├── 03-read-data-rod.strux        # read-data rod definition + usage examples
├── 04-design-notes.md            # Rationale, state-of-art comparison, open questions
├── 05-basic-rods.md              # 18 basic rods: taxonomy, knots, archetypes
├── 06-expressions.strux          # Expression type system (compiled AST form)
├── 07-manifesto-review.md        # Formal review against MANIFESTO_OBJECTIVES.md
├── 08-expression-shorthand.md    # SQL-like shorthand (source form) + EBNF grammars
├── 09-syntax-reference.md        # Compact LLM system prompt reference (~2,000 tokens)
├── 10-config-inheritance.md      # strux.context cascade: project → folder → panel defaults
├── 11-panel-shorthand.md        # Implicit snaps, drop prefixes: 9 lines / 142 tokens
└── CHANGELOG.md                  # This file
```
