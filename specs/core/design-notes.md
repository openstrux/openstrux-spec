# OpenStrux v0.4 — Design Notes

## What Changed from v0.3

### 1. Union Types (new strux form)

v0.3 had records and enums. v0.4 adds **discriminated unions** — tagged sum
types where each variant carries its own typed payload. This is the single
biggest conceptual change: it turns flat enum-based `cfg` values into
hierarchical, self-documenting type trees.

**Before (v0.3):**
```
cfg:  db    string  ["postgres", "mysql"]
arg:  host  string
arg:  port  number
arg:  tls   bool
```

**After (v0.4):**
```
cfg:  source  DataSource    // union: db.sql.postgres carries PostgresConfig
```

One cfg knot replaces four. The config is validated as a unit. The type path
resolves the adapter. Certification scopes use type paths instead of flat
string values.

### 2. AccessContext (implicit knot)

v0.3 had no concept of "who is calling this pipeline and why." Authorization
was external to the graph. v0.4 makes AccessContext an **implicit knot** —
available to every rod without explicit wiring.

This is inspired by:
- **Spark**: `SparkSession` is ambient — every operation has access to it
- **Beam**: `PipelineOptions` propagate implicitly
- **Prisma**: Middleware has access to the full request context
- **gRPC**: Metadata/context propagates through the call chain

The key insight: **if access control isn't structural, it gets skipped.** By
making it implicit and fail-closed (deny by default), we guarantee every rod
evaluates authorization before I/O.

### 3. read-data (universal source rod)

v0.3 had `read-db` — specific to databases. v0.4 has `read-data` — works with
any `DataSource` variant (streams, SQL, NoSQL). This is possible because:

1. Union types make it type-safe to handle multiple backends in one rod
2. Adapters do the actual I/O — the rod is pure logic
3. Conditional knots (`@when(cfg.source:db)`) ensure the interface adapts

### 4. Adapters (new concept)

v0.3's rod `core.py` contained both logic and I/O code. v0.4 separates them:

- **Rod core**: Pure function — authorization, field masking, error routing
- **Adapter**: I/O implementation — the thing that talks to Postgres, Kafka, etc.

Adapters are:
- Registered per union leaf type + translation target
- Versioned and certified independently of rods
- Published to the hub like any other artifact

This mirrors how Prisma has "engines" (query engine, migration engine) that
are separate from the client, and how Beam has IO connectors separate from
the pipeline definition.

---

## State of the Art Comparison

| System | What we borrow | What OpenStrux adds |
|--------|---------------|---------------------|
| **Prisma** | Schema-first, `provider` as discriminant, type-safe field access | Typed union tree (not flat string), structural authz (not middleware), multi-datasource panels |
| **Beam** | Unified batch+stream, `PCollection<T>`, `DoFn` as pure function, pluggable IO | Typed config hierarchy (not string maps), built-in access control + compliance, declarative notation |
| **Spark V2** | `format` as discriminant, pushdown predicates, catalog integration | Compile-time validated config (not `Map<String,String>`), principal/intent, cert scopes |
| **dbt** | Per-adapter profiles, `type` discriminant, env overrides | Schema validation, streams + NoSQL + APIs (not SQL-only), compliance metadata |
| **Terraform** | Provider as typed config block, state/drift detection, registry | Data pipelines (not infra), compliance-by-notation (@dp, @sec), structural access control |

---

## Key Design Decisions

### Why union types instead of inheritance?

Inheritance creates coupling. Union types are closed, exhaustive, and
pattern-matchable. When the compiler sees `DataSource`, it knows every
possible variant. There's no "unknown subclass" problem.

This also maps perfectly to how LLMs reason: "a datasource is EITHER a
stream OR a database. A database is EITHER SQL OR NoSQL." Clear branching.

### Why AccessContext is implicit, not a knot?

If it were an explicit `in:` knot, panels would need to wire it to every rod.
That's noisy and error-prone. Making it implicit (like a thread-local or
context object) means:

1. Every rod gets it automatically
2. It can't be "forgotten"
3. Scope can only narrow, never widen (enforced by the graph)
4. It's still typed and validated at compile time

### Why one read-data rod instead of read-db + read-stream + read-nosql?

Because the LOGIC is the same: authorize → resolve adapter → execute → emit.
The only thing that changes is the adapter, and that's selected by the type
path. Splitting into multiple rods would mean duplicating the access control
logic, the field masking, the error handling, the metadata emission.

Beam learned this: `ReadFromJdbc`, `ReadFromKafka`, `ReadFromMongoDB` all
have near-identical pipeline integration code. The difference is in the
source-specific config. That's exactly what our union type captures.

### Why adapters are separate from rod cores?

Separation of concerns:
- Rod core is **pure logic**: testable without infrastructure
- Adapter is **I/O**: requires real connections to test
- Different certification profiles: rod core can be "tested", adapter needs "security" certification
- Different release cadence: adapter for Postgres 17 doesn't change the rod

---

## Open Questions for v0.4

### Resolved

| Question | Resolution |
|----------|-----------|
| write-data rod | Listed in `05-basic-rods.md`. Symmetric to read-data with DataTarget union |
| AccessContext in patterns | Yes, propagates with scope narrowing at each level |
| Cross-datasource joins | Each read-data evaluates independently; merge inherits intersection |

### Open

1. **Adapter versioning**: Independent of rod versions? (Leaning yes.)
2. **Policy store**: How `policy("name")` is stored/evaluated. Hub artifact? OPA/Cedar? Both?
3. **Config inheritance interaction with cert**: Does `strux.context` config inherit cert scope? (No — cert per-component.)
