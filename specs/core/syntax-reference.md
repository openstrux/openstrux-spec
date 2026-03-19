# OpenStrux v0.5 — Compact Syntax Reference

Target: LLM system prompt context. Budget: ~2,500 tokens.

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

```
@panel name {
  @dp { record }                                      // inherits controller etc. from strux.context
  @access { purpose, operation }                       // inherits basis, scope from context
  name = rod-type { key: val, key2: val2 }             // implicit chain: reads from previous rod
  name2 = rod-type { key: val, from: other.knot }      // explicit: reads from non-default output
}
```

## 18 Basic Rods

### I/O — Data

| Rod | cfg | arg | in | out | err |
|-----|-----|-----|----|-----|-----|
| `read-data` | source: DataSource, mode: ReadMode | predicate?, fields?, limit? | — | rows/elements, meta | failure |
| `write-data` | target: DataTarget | — | rows/elements | receipt, meta | failure, reject |

ReadMode: `scan`, `lookup`, `multi_lookup`, `query`, `stream`.

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
| `window` | cfg.kind: fixed\|sliding\|session, cfg.size, in.data→out.windowed |

### Control

| Rod | Key knots |
|-----|-----------|
| `guard` | cfg.policy: PolicyRef, arg.policy (shorthand), in.data→out.allowed+out.modified, err.denied |
| `store` | cfg.backend: StateBackend, cfg.mode: get\|put\|delete\|cas\|increment, in.key+in.value?→out.result |

### Compliance

| Rod | Key knots |
|-----|-----------|
| `validate` | cfg.schema, in.data→out.valid, err.invalid |
| `pseudonymize` | cfg.algo, arg.fields, in.data→out.masked |
| `encrypt` | cfg.key_ref, arg.fields, in.data→out.encrypted |

### Topology

| Rod | Key knots |
|-----|-----------|
| `split` | arg.routes, in.data→out.{route_name}... |

## DataSource Union Tree

```
DataSource = stream (kafka, pubsub, kinesis) | db (sql: postgres/mysql/bigquery, nosql: mongodb/dynamodb/firestore)
```

Config pattern: `db.sql.postgres { host, port, db_name, tls, credentials: secret_ref{provider,path} }`.
SecretRef providers: `gcp_secret_manager`, `aws_secrets_manager`, `vault`, `env`.

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

## env() and secret_ref

```
env("VAR_NAME")                             // environment variable
secret_ref { provider: vault, path: "..." } // secret reference
```
