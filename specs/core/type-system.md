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

## 7. Persistence Annotations

`@type` records may carry persistence annotations that describe how fields map to a database schema.
These annotations are translated to Prisma directives by the generator (see `specs/modules/target-nextjs/`).

### Field-Level Annotations

Field annotations appear inline after the field type:

```
@type Proposal @timestamps {
  id:       string    @pk
  title:    string
  authorId: string
  author:   Applicant @relation(field: authorId, ref: Applicant.id, onDelete: Cascade)
  email:    string    @unique
  status:   string    @default("pending")
  note:     string    @column("internal_note") @ignore
  @@index([authorId, status])
  @@table("proposals")
}
```

| Annotation | Meaning | Prisma output |
|---|---|---|
| `@pk` | Primary key; generator infers default (`cuid` for string, `autoincrement` for int) | `@id @default(cuid())` |
| `@pk(default: uuid)` | Primary key with explicit generator | `@id @default(uuid())` |
| `@default(now)` | Timestamp default | `@default(now())` |
| `@default(true\|false\|"val"\|number)` | Scalar default | `@default(value)` |
| `@unique` | Unique constraint | `@unique` |
| `@relation(field: f, ref: M.f)` | Foreign key (owned side); inverse array auto-emitted on referenced type | `@relation(fields: [...], references: [...])` |
| `@updatedAt` | Auto-update timestamp | `@updatedAt` |
| `@column("name")` | Map field to DB column name | `@map("name")` |
| `@ignore` | Exclude from generated client | `@ignore` |

**Validator aliases (emit `W_ANNOTATION_ALIAS`, build continues):**

| Alias | Canonical |
|---|---|
| `@id` | `@pk` |
| `@map("col")` | `@column("col")` |
| `@@map("tbl")` | `@@table("tbl")` |

### Block-Level Annotations

Block annotations appear as standalone statements inside a `@type` record body:

| Annotation | Meaning | Prisma output |
|---|---|---|
| `@@index([f1, f2])` | Composite index | `@@index([f1, f2])` |
| `@@unique([f1, f2])` | Composite unique constraint | `@@unique([f1, f2])` |
| `@@table("name")` | Map type to DB table name | `@@map("name")` |
| `@opaque <content>` | Preserve unmodelled DB feature (not interpreted; emitted as comment) | `// @opaque <content>` |

### Type-Level Decorators

Decorators appear on the `@type` declaration line:

| Decorator | Meaning |
|---|---|
| `@timestamps` | Auto-inject `createdAt: date @default(now)` and `updatedAt: date @updatedAt` into the record body before code generation |
| `@sealed` | Sealed record; cannot be redefined by `.strux` source files |

### `@external` Type Modifier

`@external type Name { ... }` declares a type that exists in the database but is not owned by this Openstrux project:

- Participates in the type system (can be referenced in `@relation`, used in panels, Zod schemas generated).
- The generator emits **no `model` block** for external types.
- External types **must not** carry `@pk` (validator error: `E_EXTERNAL_PK`).
- A `W_EXTERNAL_RELATION` warning is emitted when an owned type has a `@relation` to an `@external` type.

```
@external type LegacyUser { id: string, email: string }

@type Submission {
  id:       string    @pk
  owner:    LegacyUser @relation(field: ownerId, ref: LegacyUser.id)
  ownerId:  string
}
```

### AST Node Types (IR additions, v0.6)

**`FieldAnnotation`** — union of individual annotation nodes; added to `FieldDecl`:

```typescript
type FieldAnnotation =
  | { kind: "pk";        default?: "cuid" | "uuid" | "ulid" | "autoincrement" }
  | { kind: "default";   value: "now" | string | number | boolean }
  | { kind: "unique" }
  | { kind: "relation";  field: string; ref: { model: string; field: string };
      onDelete?: "Cascade" | "SetNull" | "Restrict" | "NoAction";
      onUpdate?: "Cascade" | "SetNull" | "Restrict" | "NoAction" }
  | { kind: "updatedAt" }
  | { kind: "column";    name: string }
  | { kind: "ignore" }
```

**`TypeBlockAnnotation`** — union of block-level annotation nodes; added to `TypeRecord`:

```typescript
type TypeBlockAnnotation =
  | { kind: "index";   fields: string[] }
  | { kind: "unique";  fields: string[] }
  | { kind: "table";   name: string }
  | { kind: "opaque";  content: string }
```

**`TypeRecord` additions:**

```typescript
interface TypeRecord {
  // existing fields...
  external:    boolean;          // true if @external modifier is present
  timestamps:  boolean;          // true if @timestamps decorator is present
  annotations: TypeBlockAnnotation[];   // block-level annotations
  // FieldDecl extended:
  //   fields: (FieldDecl & { annotations: FieldAnnotation[] })[]
}
```

### Validator Rules (v0.6)

| Code | Severity | Condition |
|---|---|---|
| `E_DUPLICATE_PK` | error | More than one field with `@pk` in a record |
| `E_PK_ON_UNION` | error | `@pk` applied to a field in a `@type union` |
| `E_EXTERNAL_PK` | error | `@pk` on a field in an `@external type` |
| `E_TIMESTAMPS_DUPLICATE` | error | `@timestamps` present and `createdAt` or `updatedAt` also declared explicitly |
| `E_UNRESOLVED_RELATION_REF` | error | `@relation ref` does not resolve to an existing `@type` in scope |
| `E_MISSING_RELATION_FIELD` | error | `@relation field` does not exist on the owning type |
| `E_UPDATEDAT_TYPE_MISMATCH` | error | `@updatedAt` applied to a non-`date` field |
| `E_UNKNOWN_ANNOTATION` | error | Annotation name not in the known set and not a recognised alias |
| `W_ANNOTATION_ALIAS` | warning | Prisma-familiar alias used; build continues with canonical annotation |
| `W_EXTERNAL_RELATION` | warning | Owned type has `@relation` pointing to an `@external` type |

---

## 8. Grammar

Formal EBNF for type definitions: see [grammar.md §2](grammar.md).

---

## 9. Privacy Type System

The following types support the `private-data` standard rod and the `@privacy` decorator.
They are defined here as core types, available to any `.strux` project without explicit import.

### FieldClassification, DataCategory, Sensitivity

```
@type FieldClassification {
  field:       string,
  category:    DataCategory,
  sensitivity: Sensitivity
}

@type DataCategory = enum {
  identifying, quasi_identifying, sensitive_special,
  financial, health, biometric, genetic, political,
  religious, trade_union, sexual_orientation, criminal
}

@type Sensitivity = enum { standard, special_category, highly_sensitive }
```

`FieldClassification` is the unit of privacy tagging. Every field processed by a `private-data`
rod must have an associated `FieldClassification` — either explicit (in `cfg.fields`) or derived
from a standard data model's built-in classifications.

**Category → pseudonymization scope:**

| Category | GDPR base | GDPR + BDSG |
|---|---|---|
| `identifying` | always | always |
| `quasi_identifying` | opt-in | always |
| `financial` | always | always |
| `health`, `biometric`, `genetic`, `political`, `religious`, `trade_union`, `sexual_orientation`, `criminal`, `sensitive_special` | always | always |

**Sensitivity → encryption trigger:**

| Sensitivity | Effect |
|---|---|
| `standard` | No automatic encryption |
| `special_category` | `encryption_required` defaults to `true` (GDPR Art. 9) |
| `highly_sensitive` | `encryption_required` forced to `true` |

### RetentionPolicy, RetentionBasis

```
@type RetentionPolicy {
  duration:     string,
  basis:        RetentionBasis,
  review_cycle: Optional<string>
}

@type RetentionBasis = enum {
  legal_obligation, contract_duration, consent_withdrawal,
  legitimate_interest_review, statutory_period
}
```

`RetentionPolicy` is required for all GDPR-framework `private-data` rods (Art. 5(1)(e) storage
limitation). The `duration` field is a human-readable period string (e.g., `"5y"`, `"90d"`).
The `review_cycle` is optional and specifies how often the retention decision should be reviewed.

### PrivateData\<T\>, ProcessingMetadata

```
@type PrivateData<T> {
  data:           T,
  classification: Batch<FieldClassification>,
  processing:     ProcessingMetadata
}

@type ProcessingMetadata {
  purpose:      string,
  basis:        Optional<string>,
  retention:    Optional<RetentionPolicy>,
  consent_ref:  Optional<string>
}
```

`PrivateData<T>` is the type-level marker for personal data. When a data flow carries
`PrivateData<T>`, the compiler enforces that it passes through at least one `private-data` rod
before reaching a `write-data` or `respond` sink (see §Compile-time enforcement below).

The wrapper carries classifications alongside the data, so the `private-data` rod can derive
pseudonymization scope and encryption requirements without explicit `cfg.fields` when the input
is typed as `PrivateData<T>`.

**Compile-time enforcement:**

- `PrivateData<T>` flowing to `write-data` or `respond` without passing through `private-data` → compile error.
- `cfg.fields` on `private-data` is optional when input is `PrivateData<T>`; explicit `cfg.fields` overrides embedded classifications.
- `ProcessingMetadata.purpose` and `ProcessingMetadata.retention` are used as defaults for the rod's `purpose` and `retention` knots if those knots are not explicitly set.

**Interaction with standard data models:** When `T` is a standard type (e.g., `PersonalContact`),
the embedded `classification` is automatically populated from the standard type's built-in field
classifications. The author need not populate `classification` manually.

---

## 10. Standard Personal Data Models

A set of pre-classified `@type` definitions ships with core at
`specs/modules/types/standard/personal-data/`. These types are available without explicit import.

### Sealed types

Standard types are annotated `@sealed`. A sealed type:

- Cannot be redefined in a `.strux` source file (compile error: `E_SEALED_TYPE_REDEFINITION`)
- Cannot have fields added inline
- Can be composed into custom types as a field

```
// ✗ Compile error: PersonalContact is sealed
@type PersonalContact { email: string, twitter: string }

// ✓ Composition is always allowed
@type ExtendedContact { base: PersonalContact, linkedin: Optional<string> }
```

Custom fields added alongside standard types require explicit `FieldClassification` in
`cfg.fields` if they are personal data.

### Classification propagation through nested types

When a type includes another type as a field, the inner type's field classifications propagate
to the outer type. The propagation is transitive and follows the full nesting depth.

**Example:** `UserIdentity` includes `PersonName` and `PersonalContact`:

- The `private-data` rod processing `UserIdentity` sees all classifications from both
  `PersonName` (given_name → identifying, prefix → quasi_identifying) and
  `PersonalContact` (email → identifying) plus `UserIdentity`'s own fields.

**Propagation rules:**

1. Non-personal fields (e.g., `PersonalContact.preferred_channel`) are **not** propagated —
   they are excluded from privacy processing regardless of nesting.
2. Classifications are merged from all nested types. If the same field name appears at multiple
   levels, the innermost declaration wins.
3. Custom types that compose standard types inherit all standard type classifications. The custom
   type's own fields require explicit classification in `cfg.fields`.

### Standard type reference

| Type | Key fields | Triggers encryption? |
|---|---|---|
| `PersonName` | given_name, family_name, second_family_name, middle_name (identifying); prefix, suffix (quasi) | No |
| `PersonalContact` | email, phone, mobile (identifying) | No |
| `PostalAddress` | street (identifying); city, postal_code, country (quasi) | No |
| `UserIdentity` | PersonName + PersonalContact + date_of_birth (identifying); national_id (identifying, **highly_sensitive**) | Yes — national_id |
| `EmployeeRecord` | UserIdentity + employee_id (identifying); department, position, hire_date (quasi); salary (**financial, special_category**) | Yes — salary |
| `FinancialAccount` | iban (financial, identifying); account_holder (identifying); bic, bank_name (quasi) | No |

Full definitions: [specs/modules/types/standard/personal-data/](../modules/types/standard/personal-data/)

---

## 11. Grammar (updated — see §8)

Formal EBNF for type definitions including generic forms: see [grammar.md §2](grammar.md).
Generic type references (e.g., `PrivateData<T>`) follow the container form:

```ebnf
generic_ref = name "<" type_expr ">" ;
```

`PrivateData` is a built-in generic. User-defined generics are not supported in v0.6.
