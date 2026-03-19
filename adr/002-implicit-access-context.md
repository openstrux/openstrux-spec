# ADR-002: Implicit AccessContext

- **Status:** Accepted
- **Date:** 2026-03-19
- **Context:** v0.3 had no concept of "who is calling this pipeline and
  why." Authorization was external to the graph. If access control is not
  structural, it gets skipped — this is a well-known failure mode in
  production systems.

## Decision

AccessContext (principal + intent + scope) is an **implicit knot** —
available to every rod without explicit wiring. It is not an `in:` knot
that panels must snap to each rod.

Rules:

1. Every rod gets AccessContext automatically
2. It cannot be "forgotten" — fail-closed (empty context = deny)
3. Scope can only narrow downstream, never widen (enforced at compile time)
4. It is typed and validated at compile time

## Alternatives Considered

**Explicit `in:` knot:** Panels would need to wire AccessContext to every
rod. Noisy, error-prone, and the "forget to wire it" failure mode defeats
the purpose. Rejected.

**External middleware (Prisma/Express model):** Authorization is bolted on
outside the graph. Works in practice but is invisible to the type system,
invisible to certification, and invisible to audit. The whole point of
OpenStrux is making these concerns structural. Rejected.

**Per-rod opt-in:** Rods declare whether they need access context. Creates
a gap where I/O rods could opt out. If a rod touches data, it needs
authorization — no exceptions. Rejected.

## Consequences

- AccessContext propagates through the entire rod chain implicitly
- The `@access` block on a panel sets the initial context
- Child panels/rods inherit and can narrow but never widen scope
- The `guard` rod is the explicit evaluation point, but even rods without
  an explicit guard have AccessContext available for adapter-level checks
- Inspired by: Spark's `SparkSession` (ambient), Beam's `PipelineOptions`
  (implicit propagation), gRPC's metadata/context propagation
