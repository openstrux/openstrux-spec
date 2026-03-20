# Openstrux 0.6.0 — Syntax Reference

This is the canonical entry point for the Strux language. It is designed to be
self-sufficient: an LLM should be able to generate valid `.strux` source from
this document alone, without loading the full specification. Where deeper detail
exists, a reference link is provided — load those only when the compact rules
here are insufficient.

Openstrux is an AI-native language. Express systems as typed graphs (panels of
rods); generate executable code from source. Design goals: token-efficient
(small vocabulary, short keywords, minimal boilerplate), certified by design
(typed interfaces, version + content hash, explicit certification scope),
human-translatable on demand (deterministic builds, generated readable output),
structure-first (source is the canonical artifact, code is a derived view), and
trust built in (audit and lineage from source, not bolted on afterward).

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

Shorthand form (recommended for authoring):

```
@panel name {
  @dp { record }                                      // inherits controller etc. from strux.context
  @access { purpose, operation }                       // inherits basis, scope from context
  name = rod-type { key: val, key2: val2 }             // implicit chain: reads from previous rod
  name2 = rod-type { key: val, from: other.knot }      // explicit: reads from non-default output
}
```

Shorthand rules (verbose form remains valid — compiler normalizes both to same AST):

1. `@rod` keyword and `=` separator are optional inside `@panel`.
2. `cfg.`/`arg.` prefixes are optional — compiler resolves from rod type definition.
3. Implicit linear chain — each rod reads the previous rod's default output (no snap needed).
4. `@access` intent fields can be flattened (drop the `intent:` wrapper).

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
| split | (named routes) | data |
| write-data | receipt | rows / elements |
| receive | request | — (trigger) |
| respond | sent | data |
| call | response | request |
| guard | allowed | data |
| store | result | key |
| window | windowed | data |

Ref: [panel-shorthand.md](panel-shorthand.md) for formal implicit chain rules.

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

## Context Inheritance

Shared config lives in `strux.context` files. Panels inherit and override.

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
| `@dp` | Yes | Field-level merge, panel wins on conflict |
| `@access` | Yes | Panel can **narrow** scope, never widen (compile error) |
| `@source` | Yes (by `@name`) | `cfg.source = @production` references named source; inline fields override |
| `@target` | Yes (by `@name`) | Same as @source |
| `@ops` | Yes | Field-level merge, nearest wins |
| `@sec` | Yes | Field-level merge |
| `@cert` | **No** | Per-component only, never inherited |

Ref: [config-inheritance.md](config-inheritance.md) for cascade resolution, compile-time flattening.

## env() and secret_ref

```
env("VAR_NAME")                             // environment variable
secret_ref { provider: vault, path: "..." } // secret reference
```

---

## Grammar Essentials

Full EBNF: [grammar.md](grammar.md). Key rules for generation below.

### Lexical

```
name          = letter { letter | digit | "_" | "-" }
string        = '"' { char | escape } '"'
escape        = '\"' | '\\' | '\n' | '\t' | '\r'
number        = digit { digit } [ "." digit { digit } ]
bool          = "true" | "false"
comment       = "//" (everything until end of line)
```

Whitespace (space, tab, newline) is ignored outside strings. Statements are
line-oriented; braces delimit blocks.

### Reserved Words

Keywords — cannot be used as `name`:

```
@type @panel @context @rod @dp @access @sec @ops @cert @source @target @when
@adapter
union enum record
read-data write-data receive respond call transform filter group aggregate
merge join window guard store validate pseudonymize encrypt split
AND OR NOT IN BETWEEN IS NULL LIKE EXISTS AS CASE WHEN THEN ELSE END
HAS ANY ALL DISTINCT NULLS FIRST LAST ASC DESC COALESCE
COUNT SUM AVG MIN MAX COLLECT
true false null
env secret_ref policy snap from
```

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

Verbose and shorthand forms produce identical AST. Specifically:

- `@rod name = type { cfg.field = val }` and `name = type { field: val }` are equivalent.
- Sequential rods without `from:` produce implicit snap edges using default knots.
- Flat `@access { purpose, operation }` routes to `intent` sub-structure.

---

## Semantic Essentials

Full specification: [semantics.md](semantics.md). Key rules for generation below.

### Evaluation Order

Rods execute in **topological order** of the snap graph (not declaration order).
For linear chains these are equivalent. Data dependencies must be respected;
branches from multi-output rods (filter.match/reject, split routes) MAY run in
parallel.

### Implicit Chaining Rules

1. First rod in a panel has no implicit input (it is a source).
2. Each subsequent rod without `from:` reads the previous rod's **default output**.
3. `from: rod.knot` overrides implicit chain (reads named output knot).
4. `from: [rod1, rod2]` for multi-input rods (first = left, second = right).
5. Implicit chain follows the **default** output only (match, not reject).
   Non-default outputs require explicit `from:` on downstream rods.
6. After normalization, all chains are explicit in the IR.

### AccessContext

- `@access` is evaluated implicitly for every rod — no explicit wiring needed.
- Empty AccessContext → **deny** (fail-closed).
- Scope narrows downstream, never widens.
- `guard` rod is the explicit business-policy evaluation point.
- Narrowing is enforced at compile time.

### Error Propagation

1. If `err` knot is wired to a downstream rod → error data flows there.
2. If `err` knot is unwired → check `@ops { fallback }` or `@ops { retry }`.
3. If neither → panel fails with unhandled error. Errors are **never** silently discarded.

### Determinism

**Same source + same `snap.lock` + same `strux.context` = same compiled output.**

This applies to compiled artifacts (manifest, emitted code), not runtime behavior
of emitted code (which depends on external systems). Verbose and shorthand forms
that differ only in syntax produce identical artifacts.

### Pushdown & Fusion

Portable expressions push to any source. Source-prefixed expressions push only to
matching backends. Sequential pushable rods fuse into a single adapter call.
`fn:` references break the fusion chain (execute in-memory). Logical plan is
always preserved for audit regardless of physical fusion.

Ref: [expression-shorthand.md](expression-shorthand.md) §9 for compiler behavior, fusion examples.

---

## Full Panel Example

Self-contained verbose form (~336 tokens):

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

Same panel with context inheritance + shorthand (~142 tokens):

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

This reference is self-sufficient for generating valid `.strux` source.
Load these only when deeper detail is needed:

| Document | When to load |
|----------|-------------|
| [grammar.md](grammar.md) | Full EBNF, all productions, edge cases |
| [semantics.md](semantics.md) | Formal evaluation model, error rules, determinism proofs |
| [type-system.md](type-system.md) | Union nesting, adapter registration, type narrowing details |
| [expression-shorthand.md](expression-shorthand.md) | All expression forms with examples, compiler pushdown/fusion behavior |
| [panel-shorthand.md](panel-shorthand.md) | Shorthand derivation, token budget analysis |
| [config-inheritance.md](config-inheritance.md) | Context cascade resolution, merge/narrowing semantics, compiled output format |
| [conformance.md](conformance.md) | Conformance levels, fixture format, diagnostic codes |
| [ir.md](ir.md) | IR node structure, compiler pipeline |
| [locks.md](locks.md) | Lock file format, reproducible build guarantees |
