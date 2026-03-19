# OpenStrux v0.4 — Terminology

This glossary defines terms used throughout the OpenStrux specification.
Each term links to its authoritative spec section. Terms are normative
unless marked otherwise.

---

**AccessContext** — An implicit knot available to every rod, carrying
principal (WHO), intent (WHY), and scope (WHAT). Propagates downstream
with scope narrowing only. Fail-closed: empty context = deny. See
[access-context.strux](access-context.strux).

**adapter** — An I/O implementation registered per union leaf type and
translation target. Adapters translate rod config into target-specific
code (e.g., `PostgresConfig -> beam-python`). Separate from rod cores;
independently versioned and certified. See
[type-system.md §5](type-system.md).

**brick** — A content-addressed, versioned component in the Hub
registry. Rods, adapters, and policies are bricks. Referenced by
identity, not re-implemented.

**change package** — A directory under `changes/` containing a
proposal, design, tasks, spec deltas, and acceptance criteria for a
specification change. See `changes/README.md`.

**compiled manifest** — See **manifest**.

**conformance fixture** — A `.strux` file in `conformance/valid/`,
`conformance/invalid/`, or `conformance/golden/` used to test parser,
validator, and emitter conformance. Valid fixtures MUST parse and
typecheck clean; invalid fixtures MUST fail with the specified
diagnostic code. See [conformance.md](conformance.md).

**DataSource** — A union type representing all supported data source
backends. Top-level variants: `stream` (Kafka, PubSub, Kinesis) and
`db` (SQL, NoSQL). See
[datasource-hierarchy.strux](../modules/datasource-hierarchy.strux).

**decorator** — Metadata attached to a panel or rod that declares
intent rather than processing logic. The five decorators are `@dp`
(data protection), `@access` (authorization), `@ops` (operational
resilience), `@sec` (security), and `@cert` (certification). See
[config-inheritance.md §1](config-inheritance.md).

**discriminant** — The tag in a union variant that identifies which
branch of the union is selected. In `@type DataSource = union { stream:
StreamSource, db: DbSource }`, `stream` and `db` are discriminants.
See [type-system.md §1](type-system.md).

**emission** — The process of generating target-specific code from the
IR. Each emission target (Beam Python, TypeScript, etc.) is a
deterministic rendering of the same graph.

**knot** — A named, typed port on a rod. Five knot directions exist:
`cfg` (configuration, set at compile time), `arg` (arguments, set at
panel authoring time), `in` (data input), `out` (data output), and
`err` (error output). See [syntax-reference.md](syntax-reference.md).

**lock** — See **snap.lock**.

**manifest** (`mf.strux.json`) — The compiled, fully resolved output of
an OpenStrux compilation. Contains no inheritance, no shorthand — every
value is explicit. Used by emitters, auditors, and runtimes. See
[locks.md](locks.md).

**panel** (`@panel`) — An orchestration graph: a directed acyclic graph
of rods with attached access context and compliance metadata. A panel
is the unit of deployment and certification. See
[panel-shorthand.md](panel-shorthand.md).

**pipeline** — A panel whose source rod is `read-data` (bulk data
processing), as opposed to a service panel whose source rod is
`receive` (request/response). Informative distinction — both are
panels.

**pushdown** — Compiler optimization where portable expressions are
fused into the source adapter's native query. Sequential pushable rods
merge into a single adapter call. See
[expression-shorthand.md §9](expression-shorthand.md).

**rod** — An operation within a panel. A rod has typed knots
(cfg/arg/in/out/err), receives data, processes it, and emits data. The
18 basic rod types are defined in
[rods/overview.md](../modules/rods/overview.md).

**snap** — A connection between a rod's output knot and another rod's
input knot. Explicit form: `snap db.out.rows -> f.in.data`. Shorthand:
`from: db.rows` or implicit linear chain. See
[panel-shorthand.md](panel-shorthand.md).

**snap.lock** — A lock file capturing dependency versions, content
hashes, and resolution state. Guarantees deterministic compilation:
same source + same lock = same output. See [locks.md](locks.md).

**strux.context** (`@context`) — A configuration file at any directory
level establishing inherited defaults for all panels below it. Panels
only declare what differs from context. See
[config-inheritance.md](config-inheritance.md).

**target** — An emission platform. A target is a specific technology
stack that the compiler emits code for (e.g., `beam-python`,
`typescript`). Each target has its own adapter registry.

**type** (`@type`) — A data definition in one of three forms: record
(typed field set), enum (flat named values), or union (discriminated
tagged sum). See [type-system.md](type-system.md).

**type path** — A dot-separated path through a union tree that resolves
to a concrete type. Example: `db.sql.postgres` resolves `DataSource →
DbSource → SqlSource → PostgresConfig`. Wildcards: `db.sql.*`, `db.*`,
`*`. See [type-system.md §3](type-system.md).

**union** — A discriminated union (tagged sum type) where each variant
is a `tag: Type` pair. Closed, exhaustive, and pattern-matchable. See
[type-system.md §1](type-system.md).
