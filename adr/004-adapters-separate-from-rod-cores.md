# ADR-004: Adapters Separate from Rod Cores

- **Status:** Accepted
- **Date:** 2026-03-19
- **Context:** v0.3's rod `core.py` contained both logic and I/O code in
  the same unit. This made rods hard to test without infrastructure and
  impossible to certify independently from their I/O implementations.

## Decision

Rod cores and adapters are separate artifacts:

- **Rod core:** Pure function — authorization, field masking, error
  routing. Testable without infrastructure.
- **Adapter:** I/O implementation — the thing that talks to Postgres,
  Kafka, etc. Requires real connections to test.

Adapters are:

- Registered per union leaf type + translation target
- Versioned independently of rods
- Certified independently (rod core = "tested", adapter = "security")
- Published to the hub like any other artifact
- Pure mapping — no business logic, only config translation to the target
  framework

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

## Alternatives Considered

**Monolithic rod (logic + I/O together):** The v0.3 approach. Simpler but
untestable without infrastructure, impossible to certify the logic
independently, and forces coupled release cadence. Superseded.

**Plugin architecture (load at runtime):** Too dynamic — breaks the
determinism guarantee. Adapters must be resolved at compile time from the
type path and target. Rejected.

## Consequences

- Rod core tests are pure and fast (no infrastructure needed)
- Adapter updates (e.g., Postgres 17 support) don't require rod changes
- Different certification profiles apply: rod core can be "tested",
  adapter needs "security" certification
- Different release cadence: adapter version is independent of rod version
- Mirrors: Prisma engines (separate from client), Beam IO connectors
  (separate from pipeline definition)
