# ADR-020: Brownfield Database Support

- **Status:** Proposed
- **Date:** 2026-03-21
- **Context:** OpenStrux assumes greenfield projects: the generator emits a
  complete Prisma schema and the database is created from scratch. Real
  adoption requires entering projects with existing databases — tables,
  columns, constraints, indexes, and data already in place. Without a
  brownfield strategy, teams must either abandon their existing schema or
  abandon OpenStrux.

## Decision

OpenStrux supports brownfield database adoption through three mechanisms:
introspection, external type declarations, and migration-aware generation.

### 1. Introspection (`strux introspect`)

A CLI command reads an existing database schema and emits `.strux` type
definitions:

```
strux introspect --source postgres://... --output types/baseline.strux
```

**Mapping rules:**

| DB concept | `.strux` output |
|---|---|
| Table | `@type <PascalName> { ... }` |
| Column | Field with mapped type |
| NOT NULL | Required field (default) |
| NULL | `Optional<T>` |
| Foreign key | Reference field typed to the target type |
| Enum (PG enum or check constraint) | `@type <Name> = enum { ... }` |
| Primary key | Field with `@pk` annotation |
| Index, trigger, partial index, custom type | `@opaque` annotation (preserved, not interpreted) |

**`@opaque` annotations** capture DB features that `.strux` does not yet
model natively. They are preserved through parse → IR → emit round-trips
but the compiler does not interpret them. This prevents information loss
during introspection while keeping the type system clean.

```
@type Proposal {
  id:           string  @pk
  title:        string
  description:  Optional<string>
  submitted_at: date
  @opaque index("idx_proposals_status", [status])
  @opaque trigger("update_timestamp", before: update)
}
```

### 2. External type declarations (`@external`)

For tables that `.strux` does not own — shared tables managed by another
team, legacy tables that must not be modified, or cross-service references:

```
@external type User @table("legacy_users") {
  id:    string
  email: string
  role:  string
}
```

**Semantics:**

- The type is available to the type system: other types can reference it,
  rods can use it as input/output, schemas are generated for validation
- The generator MUST NOT emit DDL (CREATE TABLE, ALTER TABLE) for external
  types
- The generator MUST NOT include external types in the Prisma schema's
  model definitions (but MAY reference them in relation annotations)
- External types can be refreshed by re-running `strux introspect` with a
  filter

### 3. Migration-aware generation

For types that `.strux` owns (non-external), the generator emits a target
Prisma schema. The actual migration is handled by the team's chosen tool:

```
strux build → .openstrux/build/ (includes prisma/schema.prisma)
              ↓
prisma migrate diff  (or manual review, or Flyway, or raw SQL)
                 ↓
ALTER TABLE proposals ADD COLUMN reviewer_id TEXT;
```

**Key principle:** OpenStrux declares the desired state. How the team gets
from current state to desired state is their responsibility. The generator
does not execute migrations — it emits the target schema and lets existing
migration tooling handle the diff.

**Column and table mapping** annotations handle naming mismatches between
`.strux` conventions and existing DB naming:

```
@type Proposal @table("tbl_proposals") {
  id:           string
  submittedAt:  date    @column("submitted_at")
  reviewStatus: string  @column("review_status")
}
```

When no mapping annotations are present, the generator uses its default
naming convention (configurable per target adapter).

### Adoption flow

| Stage | Action |
|---|---|
| Bootstrap | `strux introspect` → generates `.strux` baseline from existing DB |
| Triage | Team reviews generated types, marks some `@external` |
| Owned types | `.strux` becomes source of truth → changes flow through `.strux` |
| New features | Written directly in `.strux` → fully generated code + schema |
| External types | Stay read-only references — refreshed by re-introspecting |

## Alternatives Considered

**Greenfield only (current approach):** Simple but limits adoption to new
projects. Most valuable software has an existing database. Superseded.

**Full bidirectional sync (DB ↔ .strux always in sync):** Two-way sync is
a distributed consistency problem. Conflict resolution is ambiguous (which
side wins?). Rejected — `.strux` owns what it owns, `@external` marks what
it doesn't.

**ORM-agnostic abstract DDL:** Emitting abstract DDL instead of
Prisma-specific schema adds a translation layer without clear benefit for
the TypeScript target. Other targets (Beam Python) will need their own
migration story anyway. Rejected for now — each target adapter handles its
own DDL format.

**Schema diffing built into OpenStrux:** Building a migration engine
duplicates mature tools (Prisma Migrate, Flyway, Alembic). OpenStrux
should declare desired state, not implement state transitions. Rejected.

## Consequences

- Adoption barrier drops significantly — teams can start with their
  existing database instead of rewriting from scratch
- The `.strux` language gains three new constructs: `@external`, `@opaque`,
  and mapping annotations (`@table`, `@column`, `@pk`)
- The toolchain gains a new command: `strux introspect`
- `@opaque` annotations create a pressure valve for DB features the type
  system doesn't model yet — but also create a risk of `.strux` files
  accumulating opaque blobs. Lint rules should warn when opaque annotations
  grow beyond a threshold.
- External types introduce a two-tier ownership model that must be clearly
  documented — "owned" types drive DDL, "external" types are read-only
  references
- The introspection command needs a database driver dependency — this is a
  toolchain concern, not a language concern, and can be implemented as a
  plugin
