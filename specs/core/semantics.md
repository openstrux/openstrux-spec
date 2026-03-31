# OpenStrux v0.4 — Evaluation Semantics

## Execution Model

OpenStrux panels do not execute directly. The compiler produces an IR
(see [ir.md](ir.md)) which an emitter renders into target-specific code
(Beam Python, TypeScript, etc.). The semantics defined here specify the
**observable behavior** that any conformant emitter MUST produce,
regardless of target platform.

A panel executes as a directed acyclic graph of rods. Each rod:

1. Receives data on its `in` knots
2. Evaluates AccessContext (implicit — see §AccessContext Evaluation)
3. Applies its operation using `cfg` and `arg` knot values
4. Emits results on its `out` knots and/or `err` knots

## Rod Chain Evaluation Order

Rods execute in **topological order** of the snap graph. For a linear
chain `a → b → c`, execution order is `a`, then `b`, then `c`. For
branching topologies (e.g., filter with match + reject outputs), all
downstream branches MAY execute in parallel — the spec does not
prescribe sequential or parallel evaluation, only that data dependencies
are respected.

**Declaration order vs evaluation order:** Declaration order determines
the implicit linear chain during normalization (see
[panel-shorthand.md](panel-shorthand.md)). After normalization, the snap
graph determines evaluation order. These are equivalent for linear
chains but may differ when `from:` overrides are used.

## Knot Data Flow

Data flows through the panel via snap edges:

```
rod_a.out.rows  →  rod_b.in.data  →  rod_b.out.match  →  rod_c.in.data
```

### Rules

1. A rod's `in` knots receive data from upstream `out` knots via snaps.
2. A rod's `out` knots emit data to downstream `in` knots via snaps.
3. A rod's `err` knots emit error/rejection data. These MUST be
   explicitly wired if consumed (no implicit chain for error outputs).
4. `cfg` knots are resolved at compile time (from source + context).
5. `arg` knots are resolved at compile time (from source expressions).
6. `in`/`out`/`err` knots carry data at evaluation time.

### Implicit vs Explicit Chaining

Per [panel-shorthand.md](panel-shorthand.md):

- **Implicit:** Sequential rods without `from:` connect via default
  knots (e.g., `filter.out.match → next.in.data`).
- **Explicit:** `from: rod.knot` overrides the implicit chain.
- **Multi-input:** `from: [rod1, rod2]` wires left and right inputs.

After normalization, all chains are explicit in the IR. The semantics
operate on the explicit snap graph only.

#### Implicit Chaining Rules 7 and 8

These rules formalize two patterns that are correct by convention but were
previously unstated. A conformant compiler MUST implement them.

**Rule 7 — Source rod starts the implicit chain:**
If the first rod in declaration order is a source rod (inKind = null, e.g.,
`read-data`, `receive`), it begins the implicit chain. The next rod's default
input knot receives from this source's default output knot. Subsequent rods
chain in declaration order unless overridden by `from:`.

**Rule 8 — Sink rod ends the implicit chain:**
If the last rod in declaration order is a sink rod (outKind = null, e.g.,
`write-data`, `respond`), it receives from the immediately preceding rod's
default output knot. A sink rod does not propagate further. No implicit chain
extends past a sink. If a non-sink rod follows a sink rod in declaration order,
it MUST have an explicit `from:` clause — a conformant compiler MUST emit an
error if this is missing.

## Type Narrowing at Evaluation Time

Union types are narrowed at compile time via type paths (see
[type-system.md §3](type-system.md)). At evaluation time:

- A rod with `cfg.source = db.sql.postgres { ... }` receives a
  `PostgresConfig` value, not a `DataSource` union.
- The adapter selected at compile time matches the narrowed type.
- No runtime type dispatch occurs — the type path fully determines
  the adapter.

`@when` conditions (e.g., `@when(cfg.source:db.sql)`) are also resolved
at compile time. Conditional knots that do not apply to the narrowed
type are removed from the IR.

## AccessContext Evaluation

AccessContext is evaluated implicitly for every rod:

1. The panel's `@access` block sets the initial AccessContext.
2. AccessContext propagates to every rod without explicit wiring.
3. Before a rod executes its operation, it evaluates access constraints:
   - If AccessContext is empty → **deny** (fail-closed).
   - If scope does not grant access to the rod's resources → **deny**.
4. The `guard` rod is the explicit policy evaluation point. It evaluates
   business-level policies (inline, hub, or external) and routes data to
   `out.allowed`, `out.modified`, or `err.denied`.
5. Scope **narrows** downstream, never widens. A rod downstream of a
   `guard` cannot have a wider scope than what the guard allowed.

### Narrowing Enforcement

Scope narrowing is enforced at compile time (see
[config-inheritance.md §5](config-inheritance.md)). At evaluation time,
the narrowing is already baked into the IR — no runtime scope checks
beyond what the emitted code implements.

## Error Propagation

### Rod `err` Knots

Each rod type defines zero or more `err` knots (e.g., `filter.err` is
not defined — rejected data goes to `out.reject`; `read-data.err`
emits `failure`).

Error propagation rules:

1. If an `err` knot is wired to a downstream rod (e.g., a DLQ
   `write-data`), error data flows there.
2. If an `err` knot is **not wired**, the behavior depends on `@ops`
   configuration:
   - With `@ops { fallback: "rod_name" }` → route to fallback rod.
   - With `@ops { retry: N }` → retry the rod up to N times.
   - Without fallback or retry → the panel fails with an unhandled
     error diagnostic.
3. A conformant implementation MUST NOT silently discard errors. Every
   error either flows to a wired downstream, triggers an `@ops`
   policy, or causes a panel failure.

### Rod-Level vs Panel-Level Errors

- **Rod-level:** Errors from a specific rod's operation (e.g., database
  connection failure, validation failure). Routed via `err` knots.
- **Panel-level:** Errors from the panel infrastructure (e.g., snap
  graph construction failure, AccessContext denial). These are reported
  as diagnostics, not via `err` knots.

## Determinism Requirement

**Same source + same lock → same output.**

A conformant implementation MUST produce identical output given:

1. Identical `.strux` source files
2. Identical `snap.lock` file
3. Identical `strux.context` files in the resolution chain

This requirement applies to the **compiled artifacts** (manifest, emitted
code), not to the runtime behavior of the emitted code (which depends on
external systems).

Specifically:

- Two source files that differ only in syntax variant (verbose vs
  shorthand) MUST produce identical compiled artifacts (see ADR-006).
- The IR is deterministic — same inputs produce the same IR.
- The manifest is deterministic — same IR produces the same
  `mf.strux.json`.
- Emitted code is deterministic — same IR + same target produces the
  same output files.

See [locks.md](locks.md) for how the lock file guarantees reproducibility.

## Standard Rod Expansion (IR Lowering)

Standard rods are expanded into sub-graphs of basic rods during IR lowering, before the IR is
handed to an emitter. This section defines the normative expansion model.

### When expansion occurs

After the panel's snap graph is resolved and all `cfg` knots are narrowed (type paths resolved),
the compiler replaces each standard rod node with its expansion sub-graph. The expansion is
deterministic: same config → same sub-graph → same lock hash.

### Expansion procedure

1. **Identify** all standard rod nodes in the IR snap graph.
2. **Resolve** the expansion rule by looking up the rod type and narrowed config.
3. **Replace** the standard rod node with the expansion sub-graph:
   - The standard rod's `in` knots are wired to the first node in the sub-graph.
   - The standard rod's `out` knots are wired from the last node in the sub-graph.
   - Error paths from expansion nodes are wired to the standard rod's `err` knots.
4. **Record** the expansion hash in the lock file: `sha256(rod_type + sorted_cfg_pairs)`.
5. **Continue** IR construction with the expanded snap graph (no standard rod nodes remain).

### Expansion rules for `private-data`

The expanded sub-graph depends on the narrowed `framework` config and the field classifications.

**Base expansion (all frameworks):**

```
in.data
  → validate   (cfg.schema: derived from fields classification)
  → pseudonymize (cfg.algo: framework-default, arg.fields: all identifying + quasi_identifying)
  [→ encrypt   (cfg.key_ref: from context, arg.fields: all special_category + highly_sensitive) — when encryption_required]
  → guard      (cfg.policy: framework lawful_basis check + access context)
  → out.protected
```

**Framework-specific expansion config:**

| Framework | pseudonymize algo default | encryption_required default | quasi_identifying masked? |
|---|---|---|---|
| `gdpr` (base) | `sha256` | `false` (unless special_category/highly_sensitive field present) | No (only if `encryption_required` true) |
| `gdpr.bdsg` | `sha256_hmac` (keyed) | `true` always | Yes (always) |

**Special category override:** If any field has `sensitivity: special_category` or `highly_sensitive`,
`encryption_required` is forced to `true` regardless of explicit config or framework default.

**BDSG additional constraint:** `pseudonymize` under `gdpr.bdsg` requires a `key_ref` in the
expansion config. A `sha256_hmac` without a key reference is a compile error.

### Determinism requirement

Two invocations of the compiler with identical `.strux` source + lock file MUST produce identical
expansion sub-graphs. Specifically:
- Field order in `cfg.fields` does not affect the expansion (fields are sorted by name before hashing).
- Framework config field order does not affect the expansion.
- The expansion hash is stable across compiler versions within the same spec version.

See [locks.md](locks.md) for how expansion hashes are stored in the lock file.

### Certification scope

A `@cert` block on a standard rod covers the **entire expansion sub-graph**. When the manifest
reports certification for a `private-data` rod, the certification covers `validate`,
`pseudonymize`, `encrypt` (if present), and `guard` as a unit — not independently.

---

## What Is NOT Specified Here

- **Target-specific behavior.** How emitted Beam Python code handles
  windowing watermarks, or how emitted TypeScript code handles HTTP
  connection pooling, is defined by the target spec
  (`specs/modules/target-beam.md`, etc.), not by core semantics.
- **Network semantics.** Latency, retry timing, circuit breaker
  thresholds — these are `@ops` decorator concerns, not evaluation
  semantics.
- **Adapter implementation.** How a Postgres adapter translates
  `PostgresConfig` into a JDBC connection is adapter-specific, not
  language-specific.
- **Runtime scheduling.** Whether rods execute in threads, processes,
  coroutines, or distributed workers is a target concern.

## Scope

This section is normative. A conformant emitter MUST produce code whose
observable behavior matches the semantics defined here.
