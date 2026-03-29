# OpenStrux v0.4 — Type System

OpenStrux has three `@type` forms: records, enums, and discriminated unions.
Unions enable typed hierarchies for datasources, service targets, and other
config structures that branch by kind.

---

## 1. Three Type Forms

| Form     | Purpose                          | Example                                |
|----------|----------------------------------|----------------------------------------|
| `record` | Typed data structure (fields)    | `@type PersonalData { ... }`          |
| `enum`   | Flat set of named values         | `@type DpBasis = enum { consent, contract }` |
| `union`  | Discriminated union (tagged sum) | `@type DataSource = union { stream: ..., db: ... }` |

The default form is `record`. Explicit keyword required for `enum` and `union`.

---

## 2. Union Types (Discriminated Unions)

### Syntax

```
@type DataSource = union {
  stream:  StreamSource
  db:      DbSource
}
```

Each variant is a `tag: Type` pair. The tag is the discriminant. The type is
any valid strux type (record, enum, union, or primitive).

### Nesting

Unions nest. The full DataSource tree is defined in
[datasource-hierarchy.strux](../modules/datasource-hierarchy.strux).

### Type Narrowing

When a `cfg` knot has a union type, setting its value narrows the type
down the tree. This is compile-time resolution:

```
cfg:  source  DataSource                       // full union
cfg.source = db.sql.postgres                   // narrows to PostgresConfig
```

After narrowing, the rod instance knows exactly which adapter it needs.

---

## 3. Type Paths

Dot-separated paths through the union tree that resolve to a concrete type:

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

---

## 4. cfg Knots with Union Types

When a `cfg` knot references a union type, the panel provides a concrete
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

The datasource config is a single typed, validated structure. The compiler
catches missing fields at build time, and the type path deterministically
selects the adapter.

---

## 5. Adapters

An adapter is the implementation behind a union variant — what actually
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
- **Registered in hub**: Shared, versioned, certified
- **Pure mapping**: No business logic — config translation to target framework only

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

## 6. DataTarget

`DataTarget` mirrors `DataSource` as the union type for write destinations.
Where `read-data` uses `cfg.source: DataSource`, `write-data` uses `cfg.target: DataTarget`.

Both are defined in [`specs/modules/datasource-hierarchy.strux`](../modules/datasource-hierarchy.strux).
The union trees are parallel but declared separately to allow independent evolution.

Type path syntax for targets is identical to sources:

```
cfg.target = stream.kafka { brokers: [...], topic: "events" }
cfg.target = db.sql.postgres { host: "...", ... }
```

Stream target paths (`stream.kafka`, `stream.pubsub`, `stream.kinesis`) have required fields:

| Adapter         | Required fields               |
|-----------------|-------------------------------|
| `stream.kafka`  | `brokers`, `topic`            |
| `stream.pubsub` | `project`, `topic`            |
| `stream.kinesis`| `region`, `stream_name`       |

---

## 7. Grammar

Formal EBNF for type definitions: see [grammar.md §2](grammar.md).
