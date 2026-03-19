# ADR-001: Union Types Over Inheritance

- **Status:** Accepted
- **Date:** 2026-03-19
- **Context:** v0.3 had records and enums but no way to express type
  hierarchies. Real-world data access requires hierarchical config: a
  datasource is either a stream or a database; a database is either SQL or
  NoSQL; SQL is either Postgres or MySQL. We needed a mechanism to express
  this.

## Decision

Use **discriminated unions** (tagged sum types) instead of inheritance or
interfaces. Each variant is a `tag: Type` pair where the tag is the
discriminant.

```
@type DataSource = union {
  stream:  StreamSource
  db:      DbSource
}
```

Unions are:

- **Closed** — the compiler knows every variant at compile time
- **Exhaustive** — pattern matching must cover all variants
- **Composable** — unions nest to form type trees
- **No subclass problem** — no unknown variants at runtime

## Alternatives Considered

**Inheritance (OOP-style):** Creates coupling, open hierarchies, and the
"unknown subclass" problem. The compiler cannot guarantee exhaustive
handling. Rejected.

**Interfaces / traits:** Too loose — they describe capability, not
identity. A datasource IS one of a known set, not "anything that implements
DataSource." Rejected.

**Flat enums + separate config:** The v0.3 approach. Works but loses type
safety — `cfg.db = "postgres"` is a string, not a typed config. Rod must
interpret the string and guess which config fields apply. Superseded.

Note: the keyword for type definitions is `@type` (not `@strux`). See
ADR-000 for the naming rationale.

## Consequences

- Union types are the sole mechanism for type hierarchies in OpenStrux
- Type paths (`db.sql.postgres`) provide dot-notation narrowing through
  the union tree
- Adapter resolution, certification scopes, and `@when` conditions all
  use type paths
- Every rod with union-typed `cfg` knots receives fully typed, validated
  config after narrowing — no runtime branching on datasource kind
- Maps naturally to how LLMs reason: "a datasource is EITHER a stream OR
  a database" — clear branching with no ambiguity
