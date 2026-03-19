# OpenStrux v0.4 — Strux Type System Refinement

## Problem Statement

v0.3 has `@strux` for records and `enum` for flat enumerations. But real-world
data access requires **type hierarchies**: a datasource is either a stream or a
database; a database is either SQL or NoSQL; SQL is either Postgres or MySQL.

This is not inheritance. It's **discriminated unions** — a pattern native to
TypeScript, Rust, Haskell, and every type system that matters. We add it to
OpenStrux as `union`.

---

## 1. Union Types (Discriminated Unions)

### Syntax

```
@strux DataSource = union {
  stream:  StreamSource
  db:      DbSource
}
```

Each variant is a `tag: Type` pair. The tag is the discriminant. The type is
any valid strux (record, enum, union, or primitive).

### Nesting

Unions nest. The full datasource tree:

```
DataSource
├── stream: StreamSource
│   ├── kafka:    KafkaConfig
│   ├── pubsub:   PubSubConfig
│   └── kinesis:  KinesisConfig
└── db: DbSource
    ├── sql: SqlSource
    │   ├── postgres: PostgresConfig
    │   ├── mysql:    MySqlConfig
    │   └── bigquery: BigQueryConfig
    └── nosql: NoSqlSource
        ├── mongodb:   MongoConfig
        ├── dynamodb:  DynamoConfig
        └── firestore: FirestoreConfig
```

This tree is **defined once** as strux types. Every rod that needs a datasource
references these types — it doesn't reinvent them.

### Type Narrowing

When a `cfg` knot has a union type, setting its value narrows the type
down the tree. This is compile-time resolution:

```
cfg:  source  DataSource                       // full union
cfg.source = db.sql.postgres                   // narrows to PostgresConfig
```

After narrowing, the rod instance knows exactly which adapter it needs. No
runtime branching on datasource kind.

### Comparison to v0.3

| v0.3 (enum)                              | v0.4 (union)                         |
|------------------------------------------|--------------------------------------|
| `@strux DbType = enum { postgres, mysql }` | `@strux SqlSource = union { postgres: PostgresConfig, mysql: MySqlConfig }` |
| Flat list of string values               | Each variant carries its own config  |
| Rod must interpret string + guess config | Rod receives typed, validated config |
| No hierarchy                             | Composable tree of types             |

---

## 2. Three Kinds of Strux

After this refinement, OpenStrux has three strux forms:

| Form     | Purpose                          | Example                                |
|----------|----------------------------------|----------------------------------------|
| `record` | Typed data structure (fields)    | `@strux PersonalData { ... }`          |
| `enum`   | Flat set of named values         | `@strux DpBasis = enum { consent, contract }` |
| `union`  | Discriminated union (tagged sum) | `@strux DataSource = union { stream: ..., db: ... }` |

The default form is `record`. Explicit keyword required for `enum` and `union`.

---

## 3. Type Paths

Union types introduce **type paths** — dot-separated paths through the union
tree that resolve to a concrete type:

```
DataSource.db.sql.postgres   →   PostgresConfig
DataSource.stream.kafka      →   KafkaConfig
DataSource.db.nosql.mongodb  →   MongoConfig
```

Type paths are used in:
- `cfg` value assignment: `cfg.source = db.sql.postgres { host: "...", port: 5432 }`
- `@when` conditions: `@when(cfg.source:db.sql)` — true for any SQL variant
- Certification scope: `{ source: "db.sql.postgres" }` or `{ source: "db.sql.*" }`
- Adapter resolution: path determines which adapter to load

### Path Matching

```
db.sql.postgres    — exact match
db.sql.*           — any SQL variant
db.*               — any database
*                  — any datasource
```

This enables certification at any level of specificity:
- "Tested with Postgres" → `{ source: "db.sql.postgres" }`
- "Tested with any SQL" → `{ source: "db.sql.*" }`

---

## 4. cfg Knots with Union Types

When a `cfg` knot references a union type, the panel must provide a concrete
path + config:

```
@rod db = read-data {
  cfg.source = db.sql.postgres {
    host:    env("DB_HOST")
    port:    5432
    db_name: "users"
    tls:     true
  }
  arg.query = "SELECT * FROM users"
}
```

This replaces the v0.3 pattern of separate `cfg.db = "postgres"` + many `arg.*`
values. The datasource config is now a single typed, validated structure.

### What moves from arg to cfg

| v0.3                                      | v0.4                                    |
|-------------------------------------------|-----------------------------------------|
| `cfg.db = "postgres"` (string)            | `cfg.source = db.sql.postgres { ... }`  |
| `arg.host`, `arg.port`, `arg.tls` (loose) | Part of `PostgresConfig` (typed)        |
| Different cfg = different cert scope      | Same: different source path = different cert scope |

Benefits:
- **Type safety**: PostgresConfig requires `host`, `port`, `db_name` — compiler catches missing fields
- **One-shot validation**: The full config is validated at compile time
- **Adapter resolution**: The type path `db.sql.postgres` deterministically selects the adapter
- **Cert scoping**: `{ source: "db.sql.postgres" }` is richer than `{ db: "postgres" }`

---

## 5. Adapters

An adapter is the implementation behind a union variant. It's what actually
connects to Postgres, reads from Kafka, etc.

### Adapter Registration

Each leaf type in a union can have a registered adapter:

```
@adapter PostgresConfig -> beam-python {
  import: "apache_beam.io.jdbc"
  class:  "ReadFromJdbc"
  map: {
    host:    "jdbc_url"     // PostgresConfig.host → ReadFromJdbc.jdbc_url
    port:    "jdbc_url"
    db_name: "jdbc_url"
    tls:     "connection_properties"
  }
}

@adapter PostgresConfig -> typescript {
  import: "@prisma/client"
  // ...
}
```

Adapters are:
- **Per union leaf**: PostgresConfig has its own, MongoConfig has its own
- **Per translation target**: Beam Python gets one adapter, TypeScript gets another
- **Registered in hub**: Shared, versioned, certified just like rods
- **Pure mapping**: No business logic — just config translation to target framework

### Adapter Resolution Chain

```
cfg.source = db.sql.postgres { ... }
          ↓
Type path: db.sql.postgres → PostgresConfig
          ↓
Target: beam-python
          ↓
Adapter: PostgresConfig -> beam-python (from hub)
          ↓
Generated code: ReadFromJdbc(jdbc_url="jdbc:postgresql://...")
```

---

## 6. Relation to State of the Art

| Framework      | What OpenStrux borrows                                    |
|----------------|-----------------------------------------------------------|
| **Prisma**     | Schema-first datasource config, type-safe client generation, provider = discriminant |
| **Apache Beam**| Unified Source/Sink abstraction, PCollection typing, DoFn = rod core |
| **Spark V2**   | DataSource API with pushdown predicates, catalog integration |
| **dbt**        | Adapter pattern per warehouse, profile = typed config      |
| **Terraform**  | Provider = discriminated union of cloud configs            |

### What OpenStrux adds

None of the above have **built-in access control + intent** at the type level.
In Prisma, authorization is middleware you bolt on. In Beam, it's external.
In OpenStrux, the `AccessContext` (principal + intent) is a first-class knot
that the rod evaluates before delegating to the adapter. See `01-access-context.strux`.

---

## 7. Grammar Extension (EBNF)

```ebnf
strux_def   = "@strux" name "=" "union" "{" union_body "}"
            | "@strux" name "=" "enum" "{" enum_body "}"
            | "@strux" name "{" record_body "}" ;

union_body  = { variant } ;
variant     = name ":" type ;

type_path   = name { "." name } ;
path_match  = type_path | type_path ".*" | "*" ;
```

---

## 8. Migration from v0.3

Existing v0.3 `@strux` records and enums are unchanged. The only new form is
`union`. Existing rods like `read-db` with `cfg.db = "postgres"` still work —
but the recommended pattern is to migrate to union-typed cfg knots.

The v0.3 `read-db` rod becomes the v0.4 `read-data` rod:
- `read-db` → `read-data` (broader: handles streams too, not just databases)
- `cfg.db string ["postgres", "mysql"]` → `cfg.source DataSource`
- Separate `arg.*` for connection → typed config inside the union variant
