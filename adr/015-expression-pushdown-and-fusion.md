# ADR-015: Expression Pushdown and Fusion Model

- **Status:** Accepted
- **Date:** 2026-03-19
- **Context:** The manifesto requires "every executable component MUST
  expose a framework-independent logic core" (P5 MUST-1) and
  "performance regressions MUST fail review" (P5 MUST-3). ADR-008
  establishes source-specific prefixes and compile-time pushdown
  validation. This ADR defines the optimization pipeline that justifies
  the P5 performance claim: how the compiler decides what pushes down,
  what fuses, and what runs in memory.

## Decision

The compiler implements a **three-phase optimization pipeline** that
transforms the logical rod graph into a physical execution plan.
The logical plan is always preserved for audit. The physical plan is
what the emitter generates.

## 1. Three Phases

```
Logical Plan → Pushdown Analysis → Fusion → Physical Plan
```

### Phase 1: Pushdown Analysis

For each rod with expression arguments, the compiler determines whether
the expression can push to the upstream source:

**Input:** Rod with compiled expression AST + upstream source type (from
snap graph traversal).

**Decision matrix:**

| Expression AST node | Upstream source | Pushable? |
|---|---|---|
| `PortableFilter` | Any | Yes |
| `PortableProjection` | Any | Yes |
| `PortableAggregation` (builtin fns) | Any | Yes |
| `PortableGroupKey` | Any | Yes |
| `PortableSort` | Any | Yes |
| `PortableJoinCond` | Any | Yes |
| `SqlFilter` (portable) | Any | Yes (treated as portable) |
| `SqlFilter` (raw) | SqlSource | Yes |
| `SqlFilter` (raw) | Non-SqlSource | **Compile error** (ADR-008) |
| `MongoFilter` | MongoSource | Yes |
| `MongoFilter` | Non-MongoSource | **Compile error** |
| `CustomFilter` | Matching source | Yes (adapter declares compatibility) |
| `CustomFilter` | Non-matching source | **Compile error** |
| `FunctionRef` | Any | **No** — breaks chain |
| `SqlAggregation` (raw) | SqlSource | Yes |
| `CustomAggregation` | Matching source | Yes |

**Output:** Each rod is annotated as `pushable`, `non-pushable`, or
`error`.

### Phase 2: Fusion

The compiler walks the rod graph in topological order and merges
consecutive pushable rods into **fusion groups**:

```
[db: read-data] → [f: filter ✓] → [t: transform ✓] → [g: group ✓] → [a: aggregate ✓]
```

Fusion rules:

1. **Start a group** at any source rod (read-data, receive)
2. **Extend the group** with each subsequent pushable rod whose
   upstream is the current group's tail
3. **Break the group** when a non-pushable rod is encountered
4. **Break the group** when the rod reads from a different upstream
   (branch join — not a linear chain)
5. **Break the group** at write-data (I/O boundary)

The fusion group becomes a single adapter call in the physical plan.

### Phase 3: Physical Plan Generation

Each fusion group is handed to the appropriate adapter for code
generation. Non-fused rods generate standalone in-memory operations.

```
LOGICAL PLAN:
  db(read-data/postgres) → f(filter) → t(transform) → g(group) → a(aggregate)

PHYSICAL PLAN:
  [FUSED] db+f+t+g+a:
    SELECT country, COUNT(*) AS total, AVG(age) AS avg_age
    FROM users
    WHERE active = true AND age >= 18
    GROUP BY country

PUSHDOWN: 4 of 4 operations pushed to source (100%)
```

## 2. Chain Breaking

When a non-pushable rod appears in the middle of a chain:

```
db(read-data) → f(filter ✓) → t(transform fn: ✗) → a(aggregate ✓)
```

Physical plan:

```
[FUSED] db+f: SELECT * FROM users WHERE active = true
[ROD]   t: fn:my-transform/core.enrich          (in-memory)
[ROD]   a: COUNT(*)                               (in-memory)
```

Note: `a` is pushable in principle but has no upstream source to fuse
with (the chain was broken by `t`). It runs in memory.

The compiler emits a suggestion:

```
INFO: t uses FunctionRef — consider portable expression for better pushdown
PUSHDOWN: 1 of 3 operations pushed to source (33%)
```

## 3. Multi-Source Panels

Panels with multiple source rods create independent fusion chains:

```
db1(read-data/postgres) → f1(filter ✓) → ──┐
                                             j(join ✓) → sink(write-data)
db2(read-data/bigquery) → f2(filter ✓) → ──┘
```

Fusion groups:

- Group 1: `db1 + f1` → single Postgres query
- Group 2: `db2 + f2` → single BigQuery query
- Group 3: `j` → in-memory (two different sources cannot fuse across
  a join unless both are the same adapter)
- `sink` → adapter call

**Same-source join optimization:** If both inputs come from the same
source (same adapter, same connection), the compiler MAY fuse the join
into a single source query. This is an optional optimization, not a
requirement.

## 4. Pushdown Metrics

The compiler computes and reports:

| Metric | Definition | Benchmark threshold |
|---|---|---|
| Pushdown rate | `pushed_ops / total_ops` | No threshold (informational) |
| Escape-hatch rate | `escape_expressions / total_expressions` | ≤10% PASS, ≤20% WARN, >20% FAIL (ADR-008) |
| Fusion group count | Number of physical operations | Fewer = better (no threshold) |
| Chain break count | Number of fusion breaks | Informational, shown in `--explain` |

These metrics are available via:

- `strux panel build --explain` — human-readable plan + metrics
- `mf.strux.json` — machine-readable metrics in the manifest
- Benchmark runner — aggregated across all panels in scope

## 5. Adapter Pushdown Contract

Adapters declare which expression AST nodes they can handle:

```json
{
  "kind": "adapter",
  "name": "PostgresConfig -> beam-python",
  "pushdown": {
    "filter": ["PortableFilter", "SqlFilter"],
    "projection": ["PortableProjection"],
    "aggregation": ["PortableAggregation", "SqlAggregation"],
    "group_key": ["PortableGroupKey"],
    "sort": ["PortableSort", "SqlSort"],
    "join": ["PortableJoinCond"]
  }
}
```

The compiler uses this declaration to determine pushability. If an
adapter doesn't declare support for a node type, the rod falls back
to in-memory execution (no error — just no pushdown).

Custom adapters can declare support for custom expression nodes:

```json
{
  "pushdown": {
    "filter": ["PortableFilter", "CustomFilter:elasticsearch"]
  }
}
```

## 6. Determinism

The optimization pipeline is **deterministic**: same logical plan +
same adapter capabilities = same physical plan. This follows from:

- Pushdown analysis depends only on expression AST types and source types
- Fusion depends only on graph topology and pushdown annotations
- No heuristics, no cost-based optimization, no statistics

Cost-based optimization (e.g., choosing between index scan and full
scan) is the adapter's responsibility at code generation time, not the
compiler's. The compiler decides **what** pushes; the adapter decides
**how**.

## 7. Conformance Cases

### Valid

- `v015-full-fusion.strux` — Linear chain of portable expressions,
  all fuse into single source query. `--explain` shows 100% pushdown.
- `v015-chain-break.strux` — Chain broken by `fn:` reference. First
  segment fuses, rest runs in memory. `--explain` shows partial pushdown.
- `v015-multi-source.strux` — Two source rods feeding a join. Each
  source chain fuses independently.

### Invalid

- `i015-prefix-mismatch.strux` — `mongo:` expression on SQL source.
  Expected: compile error (per ADR-008).

## Alternatives Considered

**Cost-based optimizer:** Uses statistics (table sizes, index
availability) to choose physical plans. Introduces non-determinism
(different stats = different plan) and requires runtime metadata.
Rejected for v0.5 — the deterministic model is simpler and auditable.
Adapters may implement cost-based decisions internally.

**No fusion (always in-memory):** Every rod runs independently, no
pushdown. Correct but slow — reads entire tables then filters in
memory. Defeats the purpose of source-specific expressions. Rejected.

**Manual fusion hints:** Author annotates which rods should fuse.
Verbose, error-prone, and the compiler has all the information to
decide automatically. Rejected.

## Consequences

- The compiler implements a three-phase pipeline: pushdown analysis,
  fusion, physical plan generation
- `strux panel build --explain` shows both logical and physical plans
- Pushdown is deterministic — no heuristics, no statistics
- Adapters declare their pushdown capabilities in their manifest
- Custom adapters can declare support for custom expression nodes
- Performance metrics (pushdown rate, escape-hatch rate, fusion count)
  are reported and available for benchmarking
- The optimization pipeline is part of the compiler, not the adapter —
  adapters only handle code generation for their fused groups
