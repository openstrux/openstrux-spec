# ADR-000: Foundational Architecture Decisions

- **Status:** Accepted
- **Date:** 2026-03-19
- **Context:** OpenStrux is a declarative language for describing data
  pipelines and backend systems with built-in compliance, access control,
  and multi-target emission. It is designed for AI-native development:
  LLMs are the primary authors, humans are reviewers and operators. Before
  individual design decisions can be evaluated, the foundational bets must
  be recorded. These are the load-bearing assumptions that every other
  decision depends on.

## The Core Problem

The single biggest waste in AI code generation is regenerating known
patterns. Every JWT middleware, every Postgres query builder, every React
form — hundreds of tokens for something written millions of times. The
underlying structure is a graph (functions call functions, data flows
through transformations, types constrain edges), but current code is
linear text generated character by character.

OpenStrux exists to eliminate this waste through a four-layer architecture
where each layer addresses a specific inefficiency.

---

## The Four Layers

### Layer 1: Brick Registry (the Hub)

Content-addressed, immutable, versioned components referenced by identity,
not re-implemented. Instead of generating a Postgres adapter from scratch,
the notation references a specific, tested, frozen implementation from the
registry.

In OpenStrux v0.5:

- **Rods** are registered components — the 18 basic rod types are the
  foundational brick set
- **Adapters** are registered per union leaf type + translation target
  (e.g., `PostgresConfig -> beam-python`)
- **Policies** are named, versioned artifacts (`policy("data-engineering-read")`)
- All registry artifacts are versioned, certified, and locked via `snap.lock`

The registry grows over time. A mature ecosystem means fewer tokens per
panel because more patterns are bricks, not inline code.

### Layer 2: Semantic Graph Notation (`.strux` syntax)

The graph structure of a system, serialized with minimal tokens. No
keywords where node type implies meaning. No boilerplate where the
compiler can infer from context. The notation describes the topology
(what connects to what) and the constraints (what types flow where),
not the implementation.

The notation has three construct families:

- **Types (`@type`)** — data definitions (record, enum, union). The type
  system, including discriminated unions and type paths.
- **Panels (`@panel`)** — orchestration graphs. A panel is a directed
  acyclic graph of rods with access context and compliance metadata.
- **Rods** — operations within panels (I/O, computation, control,
  compliance). The nodes of the graph.

Note: the type-definition keyword is `@type`, not `@strux`. "OpenStrux"
and `.strux` refer to the language and file format. `@type` refers to a
type definition within a `.strux` file. This avoids the ambiguity of
"strux" meaning both the language and a construct within it.

Key notation principles:

- **Structure-first, not code-first.** `.strux` files are declarative
  specifications, not programs. Behavior is derived by compilers and
  emitters, not executed directly.
- **Syntax variations normalize to canonical AST.** Verbose and shorthand
  forms are syntactic sugar. The compiler normalizes all accepted forms
  to the same AST before IR construction. Two source files that differ
  only in syntax variant MUST produce identical IR and identical output.
  This is the LLM-generation contract: any valid syntax variation
  produces identical results. (See ADR-006.)
- **Implicit linear chain.** Declaration order = data flow for the common
  linear case. Explicit wiring only where the graph branches. (See ADR-007.)
- **SQL-like expression shorthand.** Because every LLM and backend
  engineer already knows SQL. ~5-15 tokens per predicate vs ~30-60 for
  AST form. (See ADR-005.)
- **Discriminated unions as the type hierarchy.** Closed, exhaustive,
  pattern-matchable. No inheritance. (See ADR-001.)
- **`@type`, not `@strux`.** The keyword for type definitions is `@type`
  to avoid collision with the language name. "Strux" = the language;
  `@type` = a type definition within it.

### Layer 3: Intent Declarations (Decorators)

Instead of writing how to handle errors, validate inputs, manage state,
or enforce access control, the author declares what the constraints are.
The compiler and emitters compose the implementation from bricks.

In OpenStrux v0.5, intent declarations are decorators:

| Decorator | Intent | What it replaces |
|-----------|--------|-----------------|
| `@dp` | Data protection requirements | Hand-written GDPR compliance metadata |
| `@access` | Who can do what and why | Auth middleware, policy wiring |
| `@ops` | Retry, timeout, circuit breaker, rate limit | Resilience boilerplate |
| `@sec` | Encryption, classification, audit | Security scaffolding |
| `@cert` | Certification attestation | Manual compliance documentation |

The `!` markers from the original vision (`!auth:role>=editor`,
`!retry:3:exp`, `!latency:<200ms`) became the `@access`, `@ops`, and
`@sec` blocks. The principle is identical: **declare the contract, not
the scaffolding.**

Key intent principles:

- **Compliance by notation, not by convention.** `@dp` and `@access` are
  structural — not optional annotations. A panel without `@dp` is
  incomplete, not "lightweight."
- **Implicit AccessContext, fail-closed.** Every rod gets authorization
  context without explicit wiring. Deny by default. Scope narrows
  downstream, never widens. Enforced at compile time. (See ADR-002.)
- **Intents compose with bricks.** `@ops { retry: 3 }` doesn't generate
  inline retry code — it selects a retry strategy brick from the registry
  and wraps the rod.

### Layer 4: Human Translation Layer (Emission Targets)

Bidirectional rendering between the compact notation (Layer 2 + 3) and
human-readable code. The `.strux` graph renders into whatever humans (and
their existing tools) need.

**Forward (emit):** `.strux` → Beam Python, TypeScript, compliance
documents, visual spec, deployment manifests. Each target is a
deterministic rendering of the same graph.

**Reverse (compress, future):** Existing codebase → `.strux` notation
for efficient manipulation → re-render back. Comments, variable naming
conventions, and documentation are a human concern, not a manipulation
concern.

In OpenStrux v0.5:

- `mf.strux.json` — compiled manifest (fully resolved, human-inspectable)
- `snap.lock` — reproducibility lock (dependency versions, content hashes)
- Target emissions — Beam Python, TypeScript (MVP targets)
- `strux panel build --explain` — shows both logical and physical plans

Key translation principles:

- **Source is compact, compiled is complete.** This applies uniformly:

  | Domain | Source (compact) | Compiled (complete) |
  |--------|-----------------|-------------------|
  | Expressions | `country == "ES"` | `PortableFilter.compare {...}` |
  | Config | `@production` | Full connection details |
  | Types | `db.sql.postgres` | Full `PostgresConfig` record |
  | Panels | Implicit chain, no prefixes | Explicit snaps, prefixed knots |

- **Deterministic compilation.** Same source + same lock = same output.
  No ambient state, no non-deterministic defaults.
- **Adapters are the bridge.** Rod cores are pure logic; adapters translate
  to target-specific I/O code. Different versioning, different
  certification, different release cadence. (See ADR-004.)

---

## Cross-Cutting Decisions

### Deterministic compilation

Same source + same lock = same output. The lock file (`snap.lock`)
captures all resolution state: brick versions, adapter versions, policy
versions, content hashes. This is the reproducibility guarantee that
makes the four-layer architecture trustworthy.

### Adapters are separate from rod cores

Rod cores (pure logic) and adapters (I/O implementations) are separate
artifacts. This separation lives at the boundary between Layer 1 (bricks)
and Layer 4 (translation). (See ADR-004.)

### Source-specific pushdown

Portable expressions push to any target. Source-specific prefixes (`sql:`,
`mongo:`) allow full source-native power at the cost of portability.
Compile-time validation catches mismatches. (See ADR-008.)

---

## What This ADR Does NOT Decide

- Adapter versioning scheme (see ADR-009, proposed)
- Policy store mechanism (see ADR-010, open)
- Hub governance model and artifact format
- Specific emission targets beyond Beam Python and TypeScript
- Config inheritance interaction with certification scope (see ADR-011)
- Reverse translation (code → `.strux` compression) — future work

## Consequences

- Every subsequent ADR must be consistent with the four-layer architecture
- Language extensions must fit within the three construct families —
  `@type`, `@panel`, rods (Layer 2) — or the decorator model (Layer 3)
- New syntax forms are acceptable only if they normalize to the same AST
- Compliance metadata is mandatory, not opt-in — this limits adoption in
  contexts where compliance is irrelevant, by design
- The brick registry is a long-term investment — initial value comes from
  the 18 basic rods and their adapters; ecosystem value grows with
  community contributions
- The human translation layer is bidirectional by design, but v0.5 only
  specifies the forward direction (emit). Reverse translation is future
  work.
