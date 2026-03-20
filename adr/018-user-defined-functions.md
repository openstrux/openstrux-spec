# ADR-018: User-Defined Functions

- **Status:** Open
- **Date:** 2026-03-20
- **Context:** Strux panels express system topology as rod graphs with
  declarative decorators. Custom logic currently requires `fn:` references
  — opaque pointers to external code written in target languages (Python,
  TypeScript). This means Strux can *orchestrate* a backend but cannot
  *express* the business logic within it. The gap limits Strux to a
  pipeline DSL rather than a full backend language.

## Current State and Limitations

### What `fn:` references do today

`fn:` is a prefix in expression shorthand that creates a `FunctionRef`
IR node. It appears in any `arg.*` position:

```
arg.predicate = fn: rods/my-filter/core.apply_gdpr_rules
arg.fields    = fn: rods/my-transform/core.project_fields
arg.fn        = fn: rods/my-agg/core.custom_rollup
arg.on        = fn: rods/my-join/core.fuzzy_match
arg.policy    = fn: policies/core.evaluate_custom_policy
```

The referenced function is external to Strux — written in Python,
TypeScript, or whatever the emission target requires. The compiler
treats it as opaque: no type checking of arguments or return values,
no pushdown, no fusion.

### Limitations

1. **No type safety across the boundary.** The compiler cannot verify
   that `fn: core.enrich` accepts the upstream rod's output type or
   returns the expected downstream input type. Type errors surface at
   runtime in the emitted code, not at compile time in Strux.

2. **No pushdown or fusion.** `fn:` always breaks the optimization
   chain. A filter using `fn:` executes in-memory even when the
   function's logic could theoretically push down to the source
   (ADR-015). This is the primary performance cost of external
   functions.

3. **No portability.** A `fn:` reference points to a target-specific
   implementation. `fn: core.enrich` in a Beam Python project is a
   different function than in a TypeScript project. Multi-target
   emission requires maintaining parallel implementations.

4. **No auditability.** The function body is invisible to Strux
   tooling. `strux panel build --explain` shows `[ROD] t: fn:core.enrich`
   but cannot describe what the function does. Compliance auditors must
   inspect the external code separately.

5. **No reuse via hub.** External functions live outside the Strux
   ecosystem. They cannot be versioned, certified, or shared through
   the hub (ADR-012) the way rods and adapters can.

6. **Incomplete backend coverage.** Without native functions, Strux
   cannot express: data transformations beyond SQL-like operations,
   business rules with conditional logic, validation beyond schema
   checking, response construction for APIs, or any computation that
   does not map to the 18 built-in rod types.

## Design Space

The central question: **how much computation should Strux express
natively vs delegate to external code?**

This is a spectrum:

```
← Declarative DSL ─────────────────────────── General-purpose language →

Current Strux          Option A          Option B          Option C
(fn: refs only)     (pure transforms)  (typed functions)  (full language)
```

### Option A: Pure Transform Functions

Add `@fn` as a new top-level construct for **pure, typed, expression-
based transforms**. No I/O, no side effects, no control flow beyond
pattern matching.

```
@fn enrich_user(user: User) -> EnrichedUser {
  {
    id: user.id,
    display: COALESCE(user.nickname, user.full_name),
    tier: CASE
      WHEN user.lifetime_spend > 10000 THEN "platinum"
      WHEN user.lifetime_spend > 1000 THEN "gold"
      ELSE "standard"
    END,
    region: CASE
      WHEN user.country IN ("ES","FR","DE","IT") THEN "eu"
      WHEN user.country == "US" THEN "us"
      ELSE "other"
    END
  }
}
```

**Body language:** The existing SQL-like expression shorthand extended
with record construction (`{ field: expr }`), `CASE`/`WHEN`, and
`LET` bindings for intermediate values.

**Pros:**
- Minimal language extension — reuses existing expression syntax
- Type-safe: compiler checks input/output types at every call site
- Pushdown-eligible: if body is portable expressions, can fuse with
  upstream source
- Auditable: full function body visible to `--explain` and compliance
- Portable: same function works across all emission targets
- Aligned with "structure-first" principle — functions are data
  transforms, not imperative programs
- Hub-publishable: `@export @fn` with version and certification

**Cons:**
- Cannot express I/O, HTTP calls, file operations, or any side effect
- Cannot express loops, recursion, or complex control flow
- Pattern matching on unions requires design work (not yet specified)
- Risk of "almost but not quite" — developers hit the expressiveness
  ceiling and fall back to `fn:` anyway
- New construct family breaks the "three families" model in ADR-000
  (`@type`, `@panel`, rods) — becomes four

### Option B: Typed Functions with Statement Body

Extend Option A with a restricted imperative body: `LET` bindings,
`IF`/`ELSE`, `MATCH` on unions, `MAP`/`FILTER`/`REDUCE` over
collections. Still no I/O.

```
@fn score_application(app: Application, rules: Batch<Rule>) -> ScoredApplication {
  LET base_score = app.credit_score * 0.4 + app.income_ratio * 0.3

  LET rule_adjustments = rules
    |> FILTER(r => r.applies_to == app.category)
    |> MAP(r => r.weight * r.score_modifier)
    |> SUM()

  LET final_score = base_score + rule_adjustments

  {
    application_id: app.id,
    score: final_score,
    decision: CASE
      WHEN final_score >= 80 THEN "approve"
      WHEN final_score >= 50 THEN "review"
      ELSE "decline"
    END,
    factors: [
      { name: "credit", weight: 0.4, value: app.credit_score },
      { name: "income", weight: 0.3, value: app.income_ratio },
      { name: "rules",  weight: 1.0, value: rule_adjustments }
    ]
  }
}
```

**Pros:**
- Covers most backend business logic without external code
- Still pure — no I/O means no side effects to reason about
- Collection operations (`MAP`, `FILTER`, `REDUCE`) are natural for
  data processing
- Union `MATCH` enables type-safe branching on discriminated unions
- Sufficient for: validation logic, scoring, routing decisions, data
  enrichment, response construction

**Cons:**
- Significantly larger language surface — new syntax for `LET`, `IF`,
  `MATCH`, pipe operator, lambda expressions
- More for LLMs to learn (though all constructs are familiar from
  mainstream languages)
- Pushdown becomes harder — `LET` bindings and collection ops may not
  map to SQL
- Risk of scope creep toward a general-purpose language
- Higher implementation cost for the compiler and each emitter

### Option C: Full Imperative Language

Add full control flow, I/O capabilities, async/await, error handling.
Strux becomes a complete backend language.

**Pros:**
- No need to drop to external code, ever
- Single language for the entire backend

**Cons:**
- Destroys the core value proposition: "structure-first, code is
  derived." If Strux has a full imperative body, it IS code — not a
  structural representation of code
- Duplicates work done better by established languages (TypeScript,
  Python, Go)
- Massively increases compiler and emitter complexity
- Token efficiency degrades as function bodies grow — the compression
  ratio advantage disappears for non-structural code
- Undermines determinism: I/O in functions means side effects in the
  graph
- Goes against ADR-000 Layer 2 principle: "the notation describes the
  topology and the constraints, not the implementation"

### Option D: Keep `fn:` Only (status quo)

Do not add native functions. Improve the `fn:` boundary instead:
typed signatures for external functions, verified at compile time.

```
@extern fn enrich_user(user: User) -> EnrichedUser
  // compiler checks types at call sites
  // implementation remains external per target
```

**Pros:**
- Zero language expansion — Strux stays a pure declarative DSL
- Target languages handle computation (they're better at it)
- Smallest implementation cost
- Clearest separation of concerns

**Cons:**
- Does not solve portability (still target-specific implementations)
- Does not solve auditability (body still opaque)
- Does not solve hub reuse (external code outside Strux ecosystem)
- Does not address the "full backend" ambition
- Type safety improves (signatures checked) but function body remains
  unchecked

## Key Trade-offs

### Expressiveness vs Identity

Strux's identity is "structure-first." The more computation it
expresses natively, the more it resembles a general-purpose language
and the less distinctive its value proposition. The question is where
the line belongs — not whether one exists.

### Pushdown vs Generality

Pure expression-based functions (Option A) can potentially fuse with
source queries. Statement-based functions (Option B) largely cannot.
The more expressive the function body, the more it must execute
in-memory, reducing the optimization advantage.

### Token Efficiency

Functions that are pure expressions compress well against their
emitted equivalents (similar to rod graphs). Functions with imperative
bodies approach 1:1 token ratio with target code, losing the
compression advantage that justifies Strux.

### Compilation Complexity

Each option multiplies implementation cost across every emission
target. A `CASE`/`WHEN` expression in Option A maps cleanly to SQL,
Python, and TypeScript. A `LET` binding with pipe operator and
lambda (Option B) requires non-trivial code generation per target.

## Interaction with Other ADRs

- **ADR-000:** Functions would become a fourth construct family
  alongside `@type`, `@panel`, and rods. ADR-000 would need updating.
- **ADR-015:** Native functions could participate in pushdown/fusion
  if their body is portable expressions. This extends the optimization
  model.
- **ADR-016:** Functions need a token-efficiency story. Pure
  expression functions fit the "compact source, complete compiled"
  model. Statement-based functions may not.
- **ADR-017:** Functions need `@export`/`@use` support for cross-file
  sharing and hub publishing.

## Decision

**Not yet decided.** This ADR captures the design space for
discussion. The grant-workflow use case (v0.6.0) will identify
concrete cases where `fn:` references force dropping to external code,
providing empirical data on which option addresses real needs vs
theoretical gaps.

## References

- ADR-000: Foundational Architecture (three construct families)
- ADR-004: Adapters Separate from Rod Cores
- ADR-012: Hub Artifact Format and Governance
- ADR-015: Expression Pushdown and Fusion Model
- ADR-016: Token-Efficient Authoring Pattern
- ADR-017: Module and Package System
- `specs/core/expression-shorthand.md` — current `fn:` prefix behavior
- `specs/core/ir.md` — `FunctionRef` IR node
