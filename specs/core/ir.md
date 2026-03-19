# OpenStrux v0.4 — Intermediate Representation (IR)

## What the IR Is

The IR is the internal representation of an OpenStrux source file after
parsing and normalization, before emission to a target platform. It is
the canonical, fully resolved form of the program graph.

**Parse → Normalize → IR → Validate → Emit**

The IR is produced by the compiler after:

1. Parsing the `.strux` source (both verbose and shorthand forms)
2. Normalizing all syntax variations to canonical form (see ADR-006)
3. Resolving all `@context` inheritance (flattening config cascades)
4. Resolving all named source references (`@production` → concrete config)
5. Resolving all type paths to concrete types

The IR is consumed by:

- The **validator** — type checking, scope verification, pushdown compatibility
- The **emitter** — target-specific code generation
- The **explain** tool — logical and physical plan display

## IR Node Types

The IR mirrors the three construct families in the grammar.

### Type Nodes

| IR Node | Source Construct | Contents |
|---------|-----------------|----------|
| `TypeRecord` | `@type Name { ... }` | Name, fields (name + resolved type) |
| `TypeEnum` | `@type Name = enum { ... }` | Name, variant names |
| `TypeUnion` | `@type Name = union { ... }` | Name, variants (tag + resolved type ref) |

All type references in the IR are resolved — no unresolved names. If a
type name cannot be resolved, the compiler MUST emit a diagnostic before
IR construction.

### Panel Node

| IR Node | Source Construct | Contents |
|---------|-----------------|----------|
| `Panel` | `@panel name { ... }` | Name, resolved DP, resolved AccessContext, rod graph, snap edges |

The panel IR node contains:

- **Resolved DP** — fully merged from `@context` cascade + panel `@dp`
- **Resolved AccessContext** — fully merged, narrowing verified
- **Resolved Ops** — merged `@ops` from context + panel
- **Rod list** — ordered list of rod IR nodes
- **Snap graph** — explicit directed edges between rod knots (all
  implicit chains resolved to explicit snaps)

### Rod Nodes

| IR Node | Source Construct | Contents |
|---------|-----------------|----------|
| `Rod` | `name = rod-type { ... }` | Name, rod type, resolved cfg knots, resolved arg knots, snap edges |

Each rod IR node contains:

- **Rod type** — one of the 18 basic rod types
- **Cfg knots** — fully resolved configuration (union types narrowed
  to concrete types, `env()` references preserved as-is)
- **Arg knots** — fully resolved arguments (expression shorthand compiled
  to expression AST — see below)
- **Inbound snaps** — explicit edges from upstream rod output knots
- **Outbound snaps** — explicit edges to downstream rod input knots
- **Decorators** — resolved `@ops`, `@sec`, `@cert` for this rod

### Expression AST Nodes

Expressions (filter predicates, projections, etc.) are compiled from
shorthand to typed AST nodes within the IR. The expression AST is defined
in [expressions.strux](expressions.strux). Key node types:

| IR Node | Shorthand Source | Pushdown Target |
|---------|-----------------|-----------------|
| `PortableFilter` | `country == "ES"` | Any source |
| `SqlFilter` | `sql: id IN (SELECT ...)` | SqlSource only |
| `MongoFilter` | `mongo: {"field": {"$gt": 1}}` | MongoSource only |
| `KafkaFilter` | `kafka: header.type == "x"` | Kafka only |
| `FunctionRef` | `fn: mod/core.func` | None (in-memory) |

See [expression-shorthand.md §10](expression-shorthand.md) for the
complete shorthand-to-AST mapping.

## Union Narrowing in the IR

When a `cfg` knot has a union type and the source provides a type path,
the IR records the **narrowed type**:

```
Source:  cfg.source = db.sql.postgres { host: "...", port: 5432 }
IR:      cfg.source: NarrowedUnion {
           root_type: "DataSource"
           path: ["db", "sql", "postgres"]
           resolved_type: "PostgresConfig"
           value: { host: "...", port: 5432 }
         }
```

After narrowing, the rod's adapter can be deterministically selected
from the type path + target combination.

## Panel and Rod Representation

The IR represents a panel as a **directed acyclic graph (DAG)**:

- **Nodes** are rod IR nodes
- **Edges** are snap connections (output knot → input knot)
- All implicit linear chains have been resolved to explicit edges
- The graph MUST be acyclic (the compiler rejects cycles)

Rod declaration order is preserved in the IR for the implicit chain
resolution step, but after normalization, the snap graph is the
authoritative topology.

## AccessContext in the IR

AccessContext is attached to the panel IR node, not to individual rods.
It is available to every rod implicitly. The IR records:

- The **resolved AccessContext** (merged from context cascade + panel)
- The **narrowing chain** — if child panels or guard rods narrow scope,
  the IR records each narrowing step for audit

## IR Invariants

A valid IR MUST satisfy:

1. **All type references resolved.** No unresolved type names.
2. **All union paths resolved.** Every type path resolves to a concrete
   leaf type.
3. **All snaps resolved.** No implicit chains — every connection is an
   explicit edge between named knots.
4. **All config inherited.** No `@context` references — everything
   flattened.
5. **All expressions compiled.** No shorthand strings — every expression
   is a typed AST node.
6. **Acyclic graph.** The snap graph has no cycles.
7. **AccessContext present.** Every panel has a resolved AccessContext
   (even if it's the deny-by-default empty context).
8. **DP present.** Every panel has resolved `@dp` metadata (merged from
   context + panel).

## What the IR Does NOT Contain

- **Emission-target specifics.** The IR is target-independent. Beam
  Python code, TypeScript code, SQL queries — these are produced by the
  emitter from the IR, not stored in it.
- **Runtime state.** The IR is a compile-time artifact. No execution
  state, no connection pools, no runtime config.
- **Source syntax.** The IR does not preserve whether the source used
  verbose or shorthand form. Both normalize to the same IR.
- **Comments.** Source comments are not preserved in the IR.

## Implementation Reference

The `openstrux-core` package `packages/ast/` implements the IR node
types as TypeScript interfaces. These MUST stay in sync with this
specification. See the
[openstrux-core CLAUDE.md](../../../openstrux-core/CLAUDE.md) for the
mapping.

## Scope

This section is normative. A conformant compiler MUST produce an IR
satisfying the invariants above before proceeding to emission.
