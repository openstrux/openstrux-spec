# ADR-008: Source-Specific Prefixes with Compile-Time Pushdown Validation

- **Status:** Accepted
- **Date:** 2026-03-19
- **Context:** Portable expressions (`country == "ES"`) push to any source.
  But real-world pipelines need source-specific power: `sql: id IN (SELECT
  ...)`, `mongo: {"$text": {"$search": "..."}}`. The question was how to
  allow this without losing type safety.

## Decision

Source-specific expressions use a prefix (`sql:`, `mongo:`, `kafka:`,
`fn:`, `opa:`, `cedar:`) that declares their pushdown compatibility. The
compiler validates at compile time that the prefix matches the upstream
source type.

### Prefix Semantics

| Prefix | Pushable to | Validation |
|--------|-----------|-----------|
| (none) | Any source | Portable grammar parse |
| `sql:` | SqlSource only | Attempt portable parse first; if fails, wrap as SqlRaw |
| `mongo:` | MongoSource only | Wrap as MongoMatch |
| `kafka:` | Kafka sources only | Kafka-specific grammar parse |
| `fn:` | None (in-memory) | Create FunctionRef, breaks pushdown chain |
| `opa:` / `cedar:` | None (guard-only) | External policy reference |

### Compile-Time Validation

```
@rod f = filter {
  arg.predicate = mongo: {"country": "ES"}
  snap db.out.rows -> in.data        // db is db.sql.postgres
}
// COMPILE ERROR: MongoMatch expression on rod 'f' is not compatible
// with upstream SqlSource 'db'. Use portable syntax or sql: prefix.
```

The compiler traces the snap graph to determine the upstream source type,
then checks prefix compatibility. Mismatch = compile error, not runtime
surprise.

### Fusion Chain Analysis

Sequential pushable rods fuse into a single adapter call. The logical
plan is preserved for audit. Chain breaks when a non-pushable rod (e.g.,
`fn:` reference) is encountered.

## Alternatives Considered

**No source-specific expressions:** Forces all expressions into portable
syntax. Loses access to source-native features (subqueries, text search,
geo queries). Too limiting for real-world use. Rejected.

**Runtime source detection:** Expressions are generic; the runtime figures
out pushdown. Breaks the determinism guarantee — same expression might
push or not depending on runtime config. Also prevents compile-time error
reporting. Rejected.

**Separate rod types per source:** `sql-filter`, `mongo-filter`, etc.
Explodes the rod taxonomy and duplicates logic. The prefix approach keeps
the rod taxonomy clean while allowing source-specific power. Rejected.

## Escape-Hatch Cost Model

Source-specific prefixes and function references are **escape hatches**:
they trade portability, pushdown, and static verifiability for
source-native power. This trade-off must be visible and measurable.

### Classification

| Prefix | Escape-hatch? | Reason |
|---|---|---|
| (none) | No | Portable — fully analyzable |
| `sql:` (portable parse succeeds) | No | Portable expression in sql form |
| `sql:` (raw only) | Yes | Opaque to non-SQL targets |
| `mongo:` | Yes | Opaque to non-Mongo targets |
| `kafka:` | Yes | Opaque to non-Kafka targets |
| Custom prefix | Yes | Opaque to all other targets |
| `fn:` | Yes | Opaque to all targets, breaks pushdown |
| `opa:` / `cedar:` | Yes (policy) | Opaque policy, scope unverifiable (ADR-010) |

### Benchmark Accounting

The manifesto benchmarks define thresholds for escape-hatch usage rate:

- **PASS**: ≤10% of expressions use escape hatches
- **WARN**: >10% and ≤20%
- **FAIL**: >20%

The compiler counts escape-hatch events per panel:

```
escape_rate = escape_hatch_expressions / total_expressions
```

`strux panel build --explain` reports this rate in the pushdown summary:

```
PUSHDOWN: 4 of 5 operations pushed to source (80%)
ESCAPE HATCHES: 1 of 5 expressions (20%) — fn:my-transform/core.enrich
WARNING: escape-hatch rate 20% — at WARN threshold
```

### Certification Scope Annotation

Every escape-hatch expression MUST be covered by a `@cert` scope
annotation on the containing rod or panel. The certification scope
declares which source types the escape hatch has been tested with:

```
@rod f = filter {
  arg.predicate = sql: id IN (SELECT user_id FROM premium_users)
  @cert { scope: { source: "db.sql.*" } }
}
```

If an escape-hatch expression lacks a covering `@cert` scope, the
compiler emits `W_ESCAPE_UNCERTIFIED`. In `--locked` mode, this
promotes to `E_ESCAPE_UNCERTIFIED`.

This ensures escape hatches are:

1. **Counted** — benchmark rate is measurable
2. **Visible** — `--explain` output shows them
3. **Certified** — someone tested them with the actual target
4. **Bounded** — benchmark thresholds prevent overuse

## Consequences

- Portable expressions are the default and recommended form — they work
  everywhere
- Source-specific expressions trade portability for power — the prefix
  makes this trade-off explicit and visible
- Compile-time validation catches prefix/source mismatches before
  deployment
- Fusion optimization is deterministic and visible via
  `strux panel build --explain`
- Adding a new source type means adding a new prefix and pushdown rules,
  not changing the expression grammar
- Escape-hatch expressions are counted against benchmark thresholds
  and require certification scope coverage
- The cost model is enforced by tooling, not by convention — the
  compiler computes the rate and the cert check
