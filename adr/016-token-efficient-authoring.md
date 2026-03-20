# ADR-016: Token-Efficient Authoring Pattern

- **Status:** Open
- **Date:** 2026-03-19
- **Updated:** 2026-03-20
- **Context:** The manifesto requires token compression ratio <= 0.25
  (P1 SHOULD-1) and "source is compact, compiled is complete" is a
  foundational design principle. Config inheritance (spec:
  config-inheritance.md), panel shorthand (spec: panel-shorthand.md),
  and implicit linear chaining (ADR-007) each reduce source token count
  independently. This ADR formalizes how they combine into a normative
  authoring pattern so that `.strux` codebases achieve the target
  compression ratio consistently.

## Decision

A `.strux` codebase SHOULD be organized so that shared facts live in
`strux.context` files and each panel carries only its unique intent and
flow. The compiled manifest is where full detail belongs — redundancy
SHOULD be pushed into the compiled output rather than hand-authored
source files.

### File layout

Use a context cascade:

```
project-root/
  strux.context              # project-wide defaults
  domain-a/
    strux.context            # domain/team overrides
    pipelines/
      intake.strux           # small delta panel
      eligibility.strux      # small delta panel
  domain-b/
    strux.context
    pipelines/
      ...
```

The inheritance model is designed so project-wide `@dp`, default
`@access`, named `@source`/`@target`, and default `@ops`/`@sec` live
in context files while the panel declares only what differs.

**Root `strux.context`** — company/project-wide defaults: controller,
DPO, common policies, named infrastructure endpoints, baseline access
scope.

**Domain `strux.context`** — team or product overrides: narrower
access scope, domain-specific named sources/targets.

**One panel per use case** — only the business-specific record ID,
purpose, predicates, fields, routing, and exceptions.

### Linking strategy

Prefer named references over repeating inline config:

- Named `@source`/`@target` aliases in context files — a reference
  like `cfg.source: @production` costs only a few tokens compared with
  repeating a full inline source definition.
- Type paths like `db.sql.postgres` — dense and expressive.
- Panel-local rod names and implicit chain order (ADR-007), instead of
  explicit snap wiring everywhere.

AVOID linking by copying full connection blocks, repeated access
declarations, or repeated credential definitions into every panel.

### Panel shape

Inside a panel, use shorthand aggressively for the common case:

1. Declare rods in execution order.
2. Let the next rod consume the previous rod's default output unless
   branching is needed (ADR-007).
3. Use explicit `from` only for non-default outputs, branches, joins,
   or merges.
4. Keep access flat and direct where shorthand allows it.

### What NOT to inherit

Certification MUST NOT be inherited through context files (ADR-011).
Rod logic and snap wiring remain panel-specific by design.

The optimized split is:

- **Inherit** organizational defaults: `@dp`, `@access` defaults,
  named `@source`/`@target`, `@ops`, `@sec`.
- **Keep local and explicit**: panel logic, branching, rod-specific
  arguments, and any `@cert` declarations.

## Consequences

- Projects following this pattern should achieve token counts
  significantly below the self-contained panel equivalent. Early
  measurements (panel-shorthand.md examples) show the shorthand form
  at ~142 tokens vs ~209 tokens for the verbose form — a ~32%
  reduction from shorthand alone, before context inheritance savings.
- Context cascade adds a file-structure convention that tooling must
  resolve (implemented in `packages/config/`).
- The pattern is RECOMMENDED, not REQUIRED — simple single-file
  projects may skip the cascade without violating the spec.

## Open Questions

### Module system (→ ADR-017)

ADR-016 defines how *config* is shared across files (context cascade).
It does not define how *definitions* — types, panels, functions — are
shared. A codebase with multiple `.strux` files currently relies on
implicit project-wide scope: every `@type` defined anywhere is visible
everywhere. This works for small projects but creates ambiguity and
name collisions at scale.

ADR-017 addresses the missing module system: explicit imports,
namespacing, visibility, and hub package references. Together, ADR-016
(config sharing) and ADR-017 (definition sharing) form the complete
multi-file story.

## Validation

This ADR is **Open** pending empirical validation. The grant-workflow
use case (v0.6.0) will measure:

1. Token count of context-cascade panels vs self-contained panels
2. Token count of `.strux` source vs equivalent TypeScript
3. Whether the combined compression ratio meets the <= 0.25 target

Results will be recorded in `benchmarks/results/` and referenced when
this ADR moves to Accepted or Revised.

## References

- `specs/core/config-inheritance.md` — context cascade semantics
- `specs/core/panel-shorthand.md` — shorthand syntax and token impact
- `specs/core/syntax-reference.md` — compact LLM entry point (includes context inheritance summary)
- ADR-007: Implicit Linear Chain
- ADR-011: Certification Scope Not Inherited
- ADR-017: Module and Package System
- `docs/manifesto/MANIFESTO_OBJECTIVES.md` — P1 SHOULD-1 (token ratio)
