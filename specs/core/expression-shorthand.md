# OpenStrux v0.4 — Expression Shorthand

## Principle

Expressions have two forms:

| Form | Purpose | Who writes it | Token cost |
|---|---|---|---|
| **Source** (shorthand) | Authoring in .strux files | LLMs and humans | Low (~5-15 tokens per predicate) |
| **Compiled** (strux AST) | Machine processing, pushdown, audit | Compiler | High (~30-60 tokens per predicate) |

The shorthand is what you write. The strux AST (defined in
`expressions.strux`) is what the compiler produces. Same principle as
everywhere else in OpenStrux: compact source, verbose derived output.

## Design

SQL-like. Not because we're SQL-centric, but because:

1. Every LLM knows SQL syntax — zero learning curve
2. Every backend developer knows SQL — zero learning curve
3. It's the most token-efficient predicate notation that exists
4. It maps cleanly to portable expressions AND to source-specific ones

---

## 1. Filter Shorthand

### Portable (no prefix — default)

```
// Comparison
arg.predicate = country == "ES"
arg.predicate = age > 18
arg.predicate = score >= 0.95
arg.predicate = status != "deleted"

// Boolean logic
arg.predicate = country == "ES" AND active == true
arg.predicate = role == "admin" OR role == "superadmin"
arg.predicate = NOT archived
arg.predicate = (country == "ES" OR country == "FR") AND active == true

// IN list
arg.predicate = country IN ("ES", "FR", "DE", "IT", "PT")
arg.predicate = status NOT IN ("deleted", "suspended")

// Range
arg.predicate = age BETWEEN 18 AND 65

// Null checks
arg.predicate = email IS NOT NULL
arg.predicate = deleted_at IS NULL

// Pattern matching
arg.predicate = email LIKE "%@miempresa.es"
arg.predicate = name NOT LIKE "test_%"

// Nested field access (dot notation)
arg.predicate = address.country == "ES"
arg.predicate = metadata.tags.priority > 3

// Environment variable
arg.predicate = region == env("DEFAULT_REGION")

// EXISTS (nested field/array exists)
arg.predicate = address.postal EXISTS
```

### Source-Specific (prefixed)

```
// SQL — passed to adapter as-is, pushable to SqlSource only
arg.predicate = sql: age > 18 AND created_at > NOW() - INTERVAL '30 days'
arg.predicate = sql: id IN (SELECT user_id FROM premium_users WHERE active)
arg.predicate = sql: EXTRACT(YEAR FROM created_at) = 2026

// Mongo — JSON query document, pushable to MongoSource only
arg.predicate = mongo: {"age": {"$gt": 18}, "status": "active"}
arg.predicate = mongo: {"$text": {"$search": "openstrux"}}
arg.predicate = mongo: {"location": {"$near": {"$geometry": {"type": "Point", "coordinates": [2.17, 41.38]}}}}

// Kafka — header or partition filter
arg.predicate = kafka: header.event_type == "order_created"
arg.predicate = kafka: partition IN (0, 1, 2)

// Function reference — opaque, no pushdown
arg.predicate = fn: rods/my-filter/core.apply_gdpr_rules
```

Formal EBNF: see [grammar.md §7 — Filter](grammar.md).

---

## 2. Projection Shorthand

Used in `transform(mode=project)` and in `arg.fields` on read-data.

```
// Select fields (whitelist)
arg.fields = [id, email, address.country]

// Exclude fields (blacklist) — prefix with -
arg.fields = [*, -password_hash, -internal_id, -debug_log]

// Rename — AS keyword
arg.fields = [id, email, address.country AS country, full_name AS name]

// Computed fields — inline expressions
arg.fields = [id, email, amount * 0.21 AS iva, amount * 1.21 AS total]

// Coalesce
arg.fields = [id, COALESCE(nickname, full_name) AS display_name]

// Conditional
arg.fields = [id, CASE WHEN country == "ES" THEN "domestic" ELSE "international" END AS market]

// SQL-specific
arg.fields = sql: id, email, EXTRACT(YEAR FROM created_at) AS year, AGE(NOW(), dob) AS age

// Function reference
arg.fields = fn: rods/my-transform/core.project_fields
```

Formal EBNF: see [grammar.md §7 — Projection](grammar.md).

---

## 3. Aggregation Shorthand

Used in `aggregate` rod's `arg.fn`.

```
// Single builtin
arg.fn = COUNT(*)
arg.fn = SUM(amount)
arg.fn = AVG(score)
arg.fn = MIN(created_at)
arg.fn = MAX(price)
arg.fn = FIRST(name)
arg.fn = LAST(event_type)
arg.fn = COLLECT(tags)

// Distinct
arg.fn = COUNT(DISTINCT country)

// Multiple aggregations
arg.fn = [COUNT(*) AS total, SUM(amount) AS revenue, AVG(amount) AS avg_order]

// SQL-specific
arg.fn = sql: PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY latency) AS p95
arg.fn = sql: [COUNT(*), ARRAY_AGG(name ORDER BY created_at) AS names]
arg.fn = sql: STRING_AGG(tag, ', ' ORDER BY tag) AS all_tags

// Mongo-specific
arg.fn = mongo: {"$sum": "$amount"}

// Function reference
arg.fn = fn: rods/my-agg/core.custom_rollup
```

Formal EBNF: see [grammar.md §7 — Aggregation](grammar.md).

---

## 4. Group Key Shorthand

Used in `group` rod's `arg.key`.

```
// Single field
arg.key = country

// Composite key
arg.key = country, city

// Computed
arg.key = country, YEAR(created_at)

// SQL-specific
arg.key = sql: country, EXTRACT(MONTH FROM created_at)
arg.key = sql: ROLLUP(country, city)
arg.key = sql: CUBE(year, quarter, region)

// Function reference
arg.key = fn: rods/my-group/core.compute_bucket
```

Formal EBNF: see [grammar.md §7 — Group Key](grammar.md).

---

## 5. Join Condition Shorthand

Used in `join` rod's `arg.on`.

```
// Simple key match
arg.on = left.user_id == right.id

// Composite key
arg.on = left.user_id == right.id AND left.tenant == right.tenant

// SQL-specific
arg.on = sql: a.id = b.user_id AND a.region = b.region

// Function reference
arg.on = fn: rods/my-join/core.fuzzy_match
```

Formal EBNF: see [grammar.md §7 — Join Condition](grammar.md).

---

## 6. Sort Shorthand

Used where ordering is needed (e.g., inside aggregate, or explicit sort).

```
// Simple
arg.order = created_at DESC

// Multiple fields
arg.order = country ASC, created_at DESC

// Nulls handling
arg.order = score DESC NULLS LAST, id ASC

// SQL-specific
arg.order = sql: created_at DESC NULLS LAST, id ASC
```

Formal EBNF: see [grammar.md §7 — Sort](grammar.md).

---

## 7. Split Route Shorthand

Used in `split` rod's `arg.routes`. Each route is a named filter.

```
arg.routes = {
  eu:    address.country IN ("ES", "FR", "DE", "IT", "PT", "NL", "BE", "AT", "IE")
  us:    address.country == "US"
  latam: address.country IN ("MX", "AR", "CO", "CL", "BR")
  other: *
}
```

The `*` route is the default — catches anything not matched above.
Routes are evaluated in order; first match wins.

Formal EBNF: see [grammar.md §7 — Split Routes](grammar.md).

---

## 8. Guard Policy Shorthand

Used in `guard` rod's `arg.policy`. Extends filter syntax with
**context references** — principal, intent, scope, element.

```
// Role-based access
arg.policy = principal.roles HAS "admin"
arg.policy = principal.roles HAS ANY ("admin", "data-engineer", "dpo")
arg.policy = principal.roles HAS ALL ("viewer", "eu-region")

// Intent-based
arg.policy = intent.operation == "read"
arg.policy = intent.basis == "consent" OR intent.basis == "contract"

// Data-level (row-level security)
arg.policy = element.owner_id == principal.id
arg.policy = element.tenant == principal.attrs.tenant

// Combined
arg.policy = principal.roles HAS "analyst" AND element.country IN ("ES", "FR")
arg.policy = intent.operation == "export" AND principal.roles HAS "dpo"

// External policy engine (prefixed)
arg.policy = opa: data.authz.allow
arg.policy = cedar: Action::"read", Resource::"users"

// Function reference
arg.policy = fn: policies/core.evaluate_custom_policy
```

### Additional Operators (guard-only)

```
HAS          — collection contains value
HAS ANY      — collection contains at least one of values
HAS ALL      — collection contains all of values
```

Formal EBNF: see [grammar.md §7 — Guard Policy](grammar.md).

---

## 9. Compiler Behavior

### 9.1 Parsing

The compiler detects the expression type from syntax:

1. **No prefix** → Parse as portable shorthand grammar.
   If parse succeeds → `PortableFilter` / `PortableProjection` / etc.
   If parse fails → Compile error with position and expected token.

2. **`sql:` prefix** → Attempt to parse as portable first.
   If portable parse succeeds → mark as `SqlFilter.portable` (pushable to SQL AND other sources).
   If portable parse fails → wrap as `SqlRaw` (pushable to SQL only).

3. **`mongo:` prefix** → Wrap as `MongoMatch` (pushable to Mongo only).

4. **`kafka:` prefix** → Parse kafka-specific grammar.

5. **`fn:` prefix** → Create `FunctionRef` (no pushdown).

6. **`opa:` / `cedar:` prefix** → Create external policy reference (guard only).

### 9.2 Context-Based Pushdown Validation

After parsing, the compiler checks the snap graph:

```
@rod db = read-data { cfg.source = db.sql.postgres { ... } }
@rod f = filter {
  arg.predicate = address.country == "ES"         // portable
  snap db.out.rows -> in.data
}
```

Steps:

1. `f.in.data` snapped from `db.out.rows` → upstream source is `db.sql.postgres`
2. `f.arg.predicate` is `PortableFilter` → compatible with ANY source → ✓
3. Mark for pushdown: `f` can be fused with `db`
4. Physical plan: `SELECT * FROM users WHERE address_country = 'ES'`

If instead:

```
arg.predicate = mongo: {"country": "ES"}          // Mongo-specific
snap db.out.rows -> in.data                       // upstream is Postgres
```

1. Upstream source is `db.sql.postgres` (SqlSource)
2. Predicate is `MongoMatch` → compatible with MongoSource only
3. **COMPILE ERROR:** `MongoMatch expression on rod 'f' is not compatible with upstream SqlSource 'db'. Use portable syntax or sql: prefix.`

### 9.3 Fusion Chain Analysis

```
@rod db = read-data { cfg.source = db.sql.postgres { ... } }
@rod f = filter { arg.predicate = active == true AND age >= 18 }
@rod t = transform { arg.fields = [id, email, address.country AS country] }
@rod g = group { arg.key = country }
@rod a = aggregate { arg.fn = [COUNT(*) AS total, AVG(age) AS avg_age] }

snap db.out.rows -> f.in.data
snap f.out.match -> t.in.data
snap t.out.data -> g.in.data
snap g.out.grouped -> a.in.grouped
```

Compiler analysis:

1. `f` → PortableFilter → pushable to SQL → fuse with `db`
2. `t` → PortableProjection → pushable to SQL → extend fusion
3. `g` → PortableGroupKey → pushable to SQL → extend fusion
4. `a` → PortableAgg (COUNT, AVG = builtins) → pushable to SQL → extend fusion

Physical plan (single query):

```sql
SELECT country, COUNT(*) AS total, AVG(age) AS avg_age
FROM users
WHERE active = true AND age >= 18
GROUP BY country
```

Five logical rods → one SQL query. Logical plan preserved for audit.

### 9.4 Chain Breaking

```
@rod f = filter { arg.predicate = active == true }          // pushable
@rod t = transform { arg.fields = fn: my-transform/core.enrich }  // NOT pushable
@rod a = aggregate { arg.fn = COUNT(*) }                     // pushable (but chain broken)
```

1. `f` → pushable → fuse with upstream source
2. `t` → FunctionRef → **breaks chain** → must execute in memory
3. `a` → pushable but no upstream source to fuse with → executes in memory

Physical plan:

```
SQL: SELECT * FROM users WHERE active = true    (f fused with source)
In-memory: apply enrich function                (t executes in rod)
In-memory: count results                        (a executes in rod)
```

`strux panel build --explain` shows both plans:

```
LOGICAL PLAN:
  db(read-data/postgres) → f(filter) → t(transform) → a(aggregate)

PHYSICAL PLAN:
  [FUSED] db+f: SELECT * FROM users WHERE active = true
  [ROD]   t: fn:my-transform/core.enrich
  [ROD]   a: COUNT(*)

PUSHDOWN: 1 of 3 operations pushed to source (33%)
WARNING: t uses FunctionRef — consider portable expression for better pushdown
```

---

## 10. Relation to expressions.strux

`expressions.strux` defines the **compiled form** (strux AST).
This document defines the **source form** (shorthand).

The shorthand compiles to the AST. The AST is what appears in:

- `mf.strux.json` (compiled manifest)
- `snap.lock` (locked expressions)
- Audit outputs (lineage, Art. 30)
- `strux panel build --explain` (both forms shown)

The AST is never written by hand. It's a compiler output.

| Shorthand (source) | Compiles to (AST) |
|---|---|
| `country == "ES"` | `PortableFilter.compare { field: {path: ["country"]}, op: eq, value: LitString {value: "ES"} }` |
| `[id, email]` | `PortableProjection.fields { include: [{path:["id"]}, {path:["email"]}] }` |
| `COUNT(*)` | `PortableAgg.builtin { fn: count, field: null }` |
| `country` | `PortableGroupKey.field { path: ["country"] }` |
| `sql: NOW() - INTERVAL '30d'` | `SqlFilter.raw { clause: "NOW() - INTERVAL '30d'" }` |
| `fn: core.my_func` | `FunctionRef { module: "core", function: "my_func" }` |

---

## 11. Full Panel Example

Self-contained verbose form (27 lines, ~336 tokens). Same panel
appears in shorthand form in [panel-shorthand.md](panel-shorthand.md)
and with context inheritance in
[config-inheritance.md](config-inheritance.md).

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

27 lines. ~336 tokens. All overhead vs v0.3 is semantic: @access (trust),
typed credentials (security), expression shorthand (pushdown). No boilerplate.
