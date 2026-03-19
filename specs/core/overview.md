# OpenStrux v0.4 — Language Overview

## What OpenStrux Is

OpenStrux is a declarative language for describing data pipelines,
backend services, and compliance-aware systems. A `.strux` source file
specifies WHAT a system does — its data types, processing topology,
access constraints, and compliance metadata — while compilers and
emitters determine HOW that specification is realized in a target
platform (Apache Beam, TypeScript, etc.).

OpenStrux is designed for AI-native development: LLMs are the primary
authors, humans are reviewers and operators. The language optimizes for
token efficiency, deterministic compilation, and structural correctness
over human ergonomics like syntax highlighting or IDE autocomplete.

## Architecture: Four Layers

OpenStrux is organized into four architectural layers (see ADR-000):

1. **Brick Registry (Hub)** — content-addressed, versioned components
   (rods, adapters, policies) referenced by identity, not re-implemented.
2. **Semantic Graph Notation (`.strux` syntax)** — the graph structure
   of a system serialized with minimal tokens. This layer contains three
   construct families: types, panels, and rods.
3. **Intent Declarations (Decorators)** — `@dp`, `@access`, `@ops`,
   `@sec`, `@cert` declare constraints; the compiler and emitters compose
   the implementation.
4. **Human Translation Layer (Emission Targets)** — bidirectional
   rendering between compact notation and human-readable code.

## The Three Construct Families

Within the notation layer, OpenStrux has exactly three construct families:

**Types (`@type`)** define data structures. Three forms exist: records
(typed field sets), enums (flat named values), and unions (discriminated
tagged sums). See [type-system.md](type-system.md) for the full type
system, including type paths and union narrowing.

**Panels (`@panel`)** define orchestration graphs — directed acyclic
graphs of rods with attached access context and compliance metadata. A
panel is the unit of deployment and certification. See
[panel-shorthand.md](panel-shorthand.md) for syntax and
[config-inheritance.md](config-inheritance.md) for context cascading.

**Rods** are the operations within panels. OpenStrux defines 18 basic
rod types across six categories: I/O — Data (2), I/O — Service (3),
Computation (7), Control (2), Compliance (3), Topology (1). See
[rods/overview.md](../modules/rods/overview.md) for the full taxonomy.

## What a Valid Source File Looks Like

A `.strux` file contains one or more top-level declarations:

```
@type PersonalData {
  full_name:   string
  email:       string
  country:     string
}

@panel user-export {
  @dp { record: "RPA-2026-003" }
  @access { purpose: "export", operation: "read" }
  db = read-data { source: @production, mode: "scan" }
  f = filter { predicate: country == "ES" }
  sink = write-data { target: @analytics }
}
```

Both verbose and shorthand syntax are accepted and normalize to the same
AST (see [grammar.md](grammar.md) and ADR-006). The shorthand form is
RECOMMENDED for authoring.

## Core Design Principles

- **Structure-first:** `.strux` files are specifications, not programs.
- **Deterministic:** Same source + same lock = same output.
- **Explicit types:** Discriminated unions, not inheritance. Closed,
  exhaustive, pattern-matchable.
- **Compliance by notation:** `@dp` and `@access` are mandatory
  structural metadata, not optional annotations.
- **Source is compact, compiled is complete:** The source form optimizes
  for token efficiency; the compiled manifest (`mf.strux.json`) is fully
  resolved and self-contained.

See [design-notes.md](design-notes.md) for extended rationale.

## What OpenStrux Does NOT Do

- **It is not a programming language.** There are no loops, variables,
  or mutable state. Control flow is expressed as graph topology.
- **It does not execute directly.** A compiler emits target-specific code
  (Beam Python, TypeScript, etc.) which is then executed.
- **It does not replace infrastructure.** OpenStrux describes what a
  system does, not how it is deployed. Deployment concerns (Kubernetes,
  Terraform) are out of scope.
- **It does not invent new computation primitives.** The 18 basic rods
  are derived from established systems (Beam, Spark, Flink, Express, gRPC).
  OpenStrux adds structural compliance and access control on top.

## Scope

This specification defines OpenStrux v0.4. Normative sections use
RFC 2119 keywords (MUST, SHALL, MAY). The core spec files in this
directory define the language; module specs in `../modules/` define
rods, datasource hierarchies, and emission targets.
