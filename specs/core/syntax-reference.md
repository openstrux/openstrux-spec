# Openstrux 0.6.0 — Syntax Reference

Self-sufficient entry point — generate valid `.strux` from this document alone.
Deeper spec links provided; load only when compact rules here are insufficient.

Openstrux: AI-native language. Systems as typed graphs (panels of rods);
generate executable code from source. Goals: token-efficient, certified by
design, human-translatable, structure-first, trust built in.

---

## Type Forms

```
@type Name { field: Type, field2: Type }                    // record
@type Name = enum { val1, val2, val3 }                      // enum
@type Name = union { tag1: Type1, tag2: Type2 }             // union
```

Primitives: `string`, `number`, `bool`, `date`, `bytes`.
Containers: `Optional<T>`, `Batch<T>`, `Map<K,V>`, `Single<T>`, `Stream<T>`.
Constraint: `number [0..100]`, `string ["a","b","c"]`.

## Type Paths

Union narrowing via dot-path: `db.sql.postgres` resolves `DataSource → DbSource → SqlSource → PostgresConfig`.
Wildcards: `db.sql.*` (any SQL), `db.*` (any DB), `*` (any).

## Panel Structure

Shorthand (recommended for authoring):

```
@panel name {
  @dp { record }                                      // inherits controller etc. from strux.context
  @access { purpose, operation }                       // inherits basis, scope from context
  name = rod-type { key: val, key2: val2 }             // implicit chain: reads from previous rod
  name2 = rod-type { key: val, from: other.knot }      // explicit: reads from non-default output
}
```

Shorthand rules (verbose remains valid — both normalize to same AST):

1. `@rod` keyword optional inside `@panel` — every non-decorator statement is a rod.
   The `=` separator between name and type remains: `name = rod-type { ... }`.
2. `cfg.`/`arg.` prefixes optional — compiler resolves from rod type definition.
3. Implicit linear chain — each rod reads previous rod's default output (no snap needed).
4. `@access` intent fields can be flattened (drop the `intent:` wrapper).

Verbose and shorthand equivalence:

```
@rod f = filter { arg.predicate = ..., snap db.out.rows -> in.data }   // verbose
f = filter { predicate: ..., from: db.rows }                           // shorthand explicit
f = filter { predicate: ... }                                          // shorthand implicit chain
src = read-data { source: @production, mode: "scan", @ops { retry: 5, fallback: @backup } }  // rod-level @ops
```

## 18 Basic Rods

### I/O — Data

| Rod | cfg | arg | in | out | err |
|-----|-----|-----|----|-----|-----|
| `read-data` | source: DataSource, mode: ReadMode | predicate?, fields?, limit? | — | rows/elements, meta | failure |
| `write-data` | target: DataTarget | — | rows/elements | receipt, meta | failure, reject |

ReadMode: `scan`, `lookup`, `multi_lookup`, `query`, `stream`.
Table/collection selection via `arg.predicate` (e.g., `predicate: sql: SELECT * FROM users`).

### I/O — Service

| Rod | cfg | arg | in | out | err |
|-----|-----|-----|----|-----|-----|
| `receive` | trigger: Trigger | timeout? | — | request, context | invalid |
| `respond` | — | status?, headers? | data | sent | failure |
| `call` | target: ServiceTarget, method: CallMethod | path?, headers?, timeout? | request | response, metadata | failure, timeout |

Trigger: `http{method,path}`, `grpc{service,method}`, `event{source,topic}`, `schedule{cron?,interval?}`, `queue{source,queue}`, `manual{}`.
ServiceTarget: `http{base_url,auth?,tls}`, `grpc{host,port,proto,tls}`, `function{provider,name}`.
CallMethod: `get`, `post`, `put`, `patch`, `delete`, `unary`, `server_stream`, `invoke`.

### Computation

| Rod | Key knots |
|-----|-----------|
| `transform` | cfg.mode: map\|flat_map\|project, arg.fields/fn, in.data→out.data |
| `filter` | arg.predicate, in.data→out.match+out.reject |
| `group` | arg.key, in.data→out.grouped |
| `aggregate` | arg.fn, in.grouped→out.result |
| `merge` | in.left+in.right→out.merged |
| `join` | cfg.mode: inner\|left\|right\|outer\|cross\|lookup, arg.on, in.left+in.right→out.joined+out.unmatched |
| `window` | cfg.kind: fixed\|sliding\|session, cfg.size: duration, in.data→out.windowed |

### Control

| Rod | Key knots |
|-----|-----------|
| `guard` | cfg.policy: PolicyRef, arg.policy (shorthand), in.data→out.allowed+out.modified, err.denied |
| `store` | cfg.backend: StateBackend, cfg.mode: get\|put\|delete\|cas\|increment, in.key+in.value?→out.result |

### Compliance

| Rod | Key knots |
|-----|-----------|
| `validate` | cfg.schema: SchemaRef, in.data→out.valid, err.invalid |
| `pseudonymize` | cfg.algo, arg.fields, in.data→out.masked |
| `encrypt` | cfg.key_ref, arg.fields, in.data→out.encrypted |

SchemaRef: `@type` reference or named schema from context (e.g., `schema: UserPayload`).

### Topology

| Rod | Key knots |
|-----|-----------|
| `split` | arg.routes, in.data→out.{route_name}... |

### Default Knots (for implicit chaining)

| Rod | Default out | Default in |
|-----|------------|------------|
| read-data | rows (db) / elements (stream) | — (source) |
| filter | match | data |
| transform | data | data |
| group | grouped | data |
| aggregate | result | grouped |
| pseudonymize | masked | data |
| validate | valid | data |
| encrypt | encrypted | data |
| merge | merged | left (+ right) |
| join | joined | left (+ right) |
| split | (named routes — no default, explicit `from:` required downstream) | data |
| write-data | receipt | rows / elements |
| receive | request | — (trigger) |
| respond | sent | data |
| call | response | request |
| guard | allowed | data |
| store | result | key |
| window | windowed | data |

`window → group → aggregate` chains without explicit wiring (`windowed` is compatible with `group.in.data` and `aggregate.in.grouped`).

## Data Union Trees

```
DataSource = stream (kafka, pubsub, kinesis) | db (sql: postgres/mysql/bigquery, nosql: mongodb/dynamodb/firestore)
```

DataTarget follows the same union tree as DataSource.

Config patterns:

```
db.sql.postgres { host, port, db_name, tls, credentials: secret_ref{provider,path} }
stream.kafka { brokers, topic, credentials }
stream.pubsub { project, topic }
stream.kinesis { region, stream_name, credentials }
```

## Expression Shorthand

Expressions use SQL-like syntax. Source-specific prefixes: `sql:`, `mongo:`, `kafka:`, `fn:`, `opa:`, `cedar:`.

### Filter (arg.predicate)

```
field == "val"                              // compare (==, !=, >, >=, <, <=)
a AND b, a OR b, NOT a, (a OR b) AND c     // logic
field IN ("a","b","c"), field NOT IN (...)   // in-list
field BETWEEN 1 AND 100                     // range
field IS NULL, field IS NOT NULL            // null
field LIKE "pat%", field EXISTS             // pattern, existence
sql: id IN (SELECT ...)                     // SQL-specific (pushes to SQL only)
mongo: {"field": {"$gt": 1}}               // Mongo-specific
fn: mod/core.my_filter                      // function ref (no pushdown)
```

### Projection (arg.fields)

```
[id, email, address.country AS country]                    // select + rename
[*, -password_hash, -internal_id]                          // exclude
[id, amount * 0.21 AS iva, COALESCE(nick, name) AS disp]  // computed
```

### Aggregation (arg.fn)

```
COUNT(*), SUM(amount), AVG(score), MIN(x), MAX(x)         // single
COUNT(DISTINCT country)                                     // distinct
[COUNT(*) AS total, AVG(age) AS avg_age]                   // multi
```

### Group Key (arg.key)

```
country, YEAR(created_at)                   // field + function
```

### Join (arg.on)

```
left.user_id == right.id AND left.tenant == right.tenant
```

### Sort (arg.order)

```
created_at DESC NULLS LAST, id ASC
```

### Split Routes (arg.routes)

```
{ eu: country IN ("ES","FR","DE"), us: country == "US", other: * }
```

### Guard Policy (arg.policy)

```
principal.roles HAS "admin"                                // role check
principal.roles HAS ANY ("admin","dpo")                    // any-of
element.owner_id == principal.id                           // row-level
opa: data.authz.allow                                      // external engine
```

Rod shorthand: `g = guard { policy: opa: data.authz.allow }`.

## Pushdown Rules

Portable expressions push to any source. Prefixed push only to matching source (`sql:` → SQL, `mongo:` → Mongo). Type mismatch = compile error. Sequential pushable rods fuse into single adapter call (logical plan preserved for audit).

## Decorators

```
@dp { controller, controller_id, dpo, record, basis?, fields? }
@sec { encryption?, classification?, audit? }
@ops { retry?, timeout?, circuit_breaker?, rate_limit?, fallback? }
@cert { scope: { source: "db.sql.postgres" }, hash, version }
@access { intent: { purpose, basis, operation }, scope: policy("name") }
```

`@ops` field types:

```
retry: number                                // max attempts
timeout: duration                            // "30s", "5m"
fallback: @name | rod-name                   // named source or rod on exhaustion
circuit_breaker: { threshold: number, window: duration }
rate_limit: { max: number, window: duration }
```

## Context Inheritance

Shared config in `strux.context` files. Panels inherit and override.

```
project-root/
  strux.context              # project-wide: @dp, named @source/@target, @ops, @sec
  domain-a/
    strux.context            # team overrides (narrower @access, domain sources)
    pipelines/
      panel.strux            # only unique intent, logic, routing
```

### `strux.context` syntax

```
@context {
  @dp { controller: "Acme", controller_id: "B-123", dpo: "dpo@acme.com" }
  @access { intent: { basis: "legitimate_interest" }, scope: policy("default-read") }
  @source production = db.sql.postgres { host: env("DB_HOST"), port: 5432, ... }
  @target analytics = db.sql.bigquery { project: "analytics", location: "EU", ... }
  @ops { retry: 3, timeout: "30s" }
}
```

### Inheritance rules

| Block | Inheritable | Merge behavior |
|-------|------------|----------------|
| `@dp` | Yes | Field-level merge, panel wins |
| `@access` | Yes | Panel can **narrow** scope, never widen (compile error) |
| `@source` | Yes (by `@name`) | `cfg.source = @production` references named source; inline fields override |
| `@target` | Yes (by `@name`) | Same as @source |
| `@ops` | Yes | Field-level merge, nearest wins |
| `@sec` | Yes | Field-level merge |
| `@cert` | **No** | Per-component only, never inherited |

## Built-in References and Credentials

```
@name                                       // named @source/@target from strux.context
@name { field: override }                   // resolve + override fields inline
env("VAR_NAME")                             // environment variable
secret_ref { provider: vault, path: "..." } // secret reference
policy("name")                              // named policy (hub, OPA, or Cedar); compile-time ref
adc {}                                      // GCP Application Default Credentials (empty record)
```

SecretRef providers: `gcp_secret_manager`, `aws_secrets_manager`, `vault`, `env`.

---

## Grammar Essentials

### Lexical

```
name          = letter { letter | digit | "_" | "-" }
string        = '"' { char | escape } '"'
escape        = '\"' | '\\' | '\n' | '\t' | '\r'
number        = digit { digit } [ "." digit { digit } ]
bool          = "true" | "false"
duration      = number ("s" | "m" | "h" | "d")               // "30s", "5m", "24h"
comment       = "//" (everything until end of line)
```

Whitespace ignored outside strings. Braces delimit blocks.

### Reserved Words

Keywords — cannot be used as `name`:

```
@type @panel @context @rod @dp @access @sec @ops @cert @source @target
@when @adapter                                                           // reserved (type-system, semantics)
union enum record
read-data write-data receive respond call transform filter group aggregate
merge join window guard store validate pseudonymize encrypt split
AND OR NOT IN BETWEEN IS NULL LIKE EXISTS AS CASE WHEN THEN ELSE END
HAS ANY ALL DISTINCT NULLS FIRST LAST ASC DESC COALESCE
COUNT SUM AVG MIN MAX COLLECT
true false null
env secret_ref policy snap from
```

Type scope notes: `StateBackend` — per adapter ([type-system.md](type-system.md)).
`PolicyRef` — hub or policy store ([design-notes.md](design-notes.md) §5).
`COLLECT` — reserved for future grouped collection. `CASE/WHEN/THEN/ELSE/END` — reserved for conditional expressions.

### Operator Precedence (highest to lowest)

| Precedence | Operators | Associativity |
|-----------|-----------|---------------|
| 1 | `()` grouping | — |
| 2 | unary `NOT`, unary `-` | right |
| 3 | `*` `/` `%` | left |
| 4 | `+` `-` | left |
| 5 | `==` `!=` `>` `>=` `<` `<=` | non-associative |
| 6 | `IN` `NOT IN` `BETWEEN` `IS` `LIKE` `EXISTS` `HAS` | non-associative |
| 7 | `AND` | left |
| 8 | `OR` | left |

### Normalization

Verbose and shorthand produce identical AST:

- `@rod name = type { cfg.field = val }` ≡ `name = type { field: val }`.
- Rods without `from:` → implicit snap edges via default knots.
- Flat `@access { purpose, operation }` → routes to `intent` sub-structure.

---

## Semantic Essentials

### Evaluation Order

Rods execute in **topological order** of the snap graph (not declaration order).
For linear chains these are equivalent. Branches from multi-output rods
(filter.match/reject, split routes) MAY run in parallel.

### Implicit Chaining Rules

1. First rod in a panel has no implicit input (it is a source).
2. Each subsequent rod without `from:` reads the previous rod's **default output**.
3. `from: rod.knot` overrides implicit chain (reads named output knot).
4. `from: [rod1, rod2]` for multi-input rods — first = left, second = right.
5. Implicit chain follows the **default** output only (match, not reject).
   Non-default outputs require explicit `from:` on downstream rods.
6. After normalization, all chains are explicit in the IR.
7. Source rods (`read-data`, `receive`, `in: —`) always start a new chain
   regardless of position — they never consume the previous rod's output.
8. A rod with explicit `from:` does not advance the implicit chain. The next
   rod without `from:` chains from the last implicit-chain rod, not the branch.

`from: rod.knot` resolves across `out` and `err` namespaces — knot names are
unique per rod, so the namespace prefix is dropped in shorthand.

Multi-input example:

```
users = read-data { source: @production, mode: "scan" }
orders = read-data { source: @production, mode: "scan" }
j = join { mode: "inner", on: left.user_id == right.id, from: [users, orders] }
//   ↑ normalizes to: snap users.out.rows -> in.left, snap orders.out.rows -> in.right
```

### AccessContext

- `@access` evaluated implicitly for every rod — no explicit wiring needed.
- Empty AccessContext → **deny** (fail-closed).
- Scope narrows downstream, never widens. Enforced at compile time.
- `guard` rod is the explicit business-policy evaluation point.

### Error Propagation

1. If `err` knot is wired to a downstream rod → error data flows there.
2. If unwired → apply `@ops { retry }` first, then `@ops { fallback }` on exhaustion.
3. If neither → panel fails with unhandled error. Errors **never** silently discarded.

`write-data` has two error knots: `failure` (transport/system, follows cascade
above) and `reject` (data-level, e.g. schema mismatch). `reject` is a typed
branch, not a failure channel — wire it like any non-default output (typically DLQ).

### Determinism

**Same source + same `snap.lock` = same compiled output.** Lock captures context
chain hashes, so `strux.context` is pinned. Applies to compiled artifacts, not
runtime behavior. Verbose and shorthand produce identical artifacts.

### Pushdown & Fusion

See Pushdown Rules above. `fn:` references break the fusion chain (execute
in-memory). Logical plan always preserved for audit regardless of physical fusion.

---

## Full Panel Example

Verbose form (~336 tokens):

```
@panel user-analytics {
  @dp { controller: "MiEmpresa SL", controller_id: "ES-B12345678", dpo: "dpo@miempresa.es", record: "RPA-2026-003" }
  @access { intent: { purpose: "geo_segmentation", basis: "legitimate_interest", operation: "read" }, scope: policy("data-engineering-read") }
  @rod db = read-data {
    cfg.source = db.sql.postgres {
      host: env("DB_HOST"), port: 5432, db_name: "production", tls: true
      credentials: secret_ref { provider: gcp_secret_manager, path: "projects/mi/secrets/pg" }
    }
    cfg.mode = "scan"
  }
  @rod f = filter {
    arg.predicate = address.country IN ("ES", "FR", "DE") AND deleted_at IS NULL
    snap db.out.rows -> in.data
  }
  @rod p = pseudonymize {
    cfg.algo = "sha256", arg.fields = ["full_name", "email", "national_id"]
    snap f.out.match -> in.data
  }
  @rod sink = write-data {
    cfg.target = db.sql.bigquery { project: "miempresa-analytics", dataset: "eu_users", location: "EU", credentials: adc {} }
    snap p.out.masked -> in.rows
  }
  @rod dlq = write-data {
    cfg.target = stream.pubsub { project: "miempresa", topic: "dlq" }
    snap f.out.reject -> in.elements
  }
}
```

With context inheritance + shorthand (~142 tokens):

```
@panel user-analytics {
  @dp { record: "RPA-2026-003" }
  @access { purpose: "geo_segmentation", operation: "read" }
  db = read-data { source: @production, mode: "scan" }
  f = filter { predicate: address.country IN ("ES", "FR", "DE") AND deleted_at IS NULL }
  p = pseudonymize { algo: "sha256", fields: ["full_name", "email", "national_id"] }
  sink = write-data { target: @analytics { dataset: "eu_users" } }
  dlq = write-data { target: @dlq, from: f.reject }
}
```

---

## Specification Map

This reference is self-sufficient for generation. Load only when deeper detail is needed:

| Document | When to load |
|----------|-------------|
| [grammar.md](grammar.md) | Full EBNF, all productions, edge cases |
| [semantics.md](semantics.md) | Formal evaluation model, error rules, determinism proofs |
| [type-system.md](type-system.md) | Union nesting, adapter registration, type narrowing details |
| [expression-shorthand.md](expression-shorthand.md) | All expression forms, compiler pushdown/fusion behavior |
| [panel-shorthand.md](panel-shorthand.md) | Shorthand derivation, token budget analysis |
| [config-inheritance.md](config-inheritance.md) | Context cascade, merge/narrowing semantics, compiled output format |
| [conformance.md](conformance.md) | Conformance levels, fixture format, diagnostic codes |
| [ir.md](ir.md) | IR node structure, compiler pipeline |
| [locks.md](locks.md) | Lock file format, reproducible build guarantees |
