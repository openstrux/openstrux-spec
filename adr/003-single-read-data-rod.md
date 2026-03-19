# ADR-003: Single read-data Rod Instead of Per-Source Rods

- **Status:** Accepted
- **Date:** 2026-03-19
- **Context:** v0.3 had `read-db` — specific to databases. But pipelines
  also read from streams, APIs, and files. The question was whether to add
  `read-stream`, `read-nosql`, etc. or unify into one rod.

## Decision

A single `read-data` rod handles all `DataSource` variants (streams, SQL,
NoSQL). The union type on `cfg.source` determines which adapter is loaded.
The rod logic (authorize → resolve adapter → execute → emit) is identical
regardless of source.

## Alternatives Considered

**Per-source rods (`read-db`, `read-stream`, `read-nosql`):** Duplicates
the access control logic, field masking, error handling, and metadata
emission across every rod. Beam learned this lesson — `ReadFromJdbc`,
`ReadFromKafka`, `ReadFromMongoDB` all have near-identical pipeline
integration code. The difference is only in source-specific config, which
is exactly what the union type captures. Rejected.

**Generic `io` rod with mode parameter:** Too loose — conflates reads and
writes, sources and services. The distinction between `read-data` (bulk
data retrieval) and `call` (service invocation) is semantically important.
Rejected.

## Consequences

- `read-data` is the universal source rod for bulk data
- `write-data` is the symmetric universal sink rod
- `call` remains separate for service-to-service communication (different
  semantics: see rods/overview.md for the distinction)
- Adapter selection is fully determined by the type path on `cfg.source`
- Adding a new datasource means adding a new union variant and adapter,
  not a new rod type
