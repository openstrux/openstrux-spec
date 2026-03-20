# ADR-017: Module and Package System

- **Status:** Open
- **Date:** 2026-03-20
- **Context:** ADR-016 defines how configuration is shared across files
  via `strux.context` cascade. It does not address how *definitions* —
  types, panels, and (future) functions — are shared. Currently, all
  `.strux` files in a project share an implicit global scope: every
  `@type` is visible everywhere, with no import/export mechanism, no
  namespacing, and no way to reference definitions from external
  packages. This works for single-file examples and small projects but
  creates three problems at scale: (1) name collisions between teams or
  domains, (2) no visibility control (every internal type is public),
  and (3) no versioned dependency on shared type libraries or hub
  packages. Academic work on DSL modularity (Erdweg et al., "Language
  Composition Untangled") and projectional editing (Voelter et al.)
  confirms that explicit module boundaries reduce composition errors and
  improve tooling support.

## Decision

Strux introduces a module system with three constructs: `@use` for
imports, `@export` for visibility, and package identifiers for hub
dependencies. The design prioritizes token efficiency (aligned with
ADR-016) and compile-time resolution (no runtime module loading).

### File as module

Each `.strux` file is a module. Its module identity is derived from its
file path relative to the project root (the directory containing the
root `strux.context`):

```
project-root/
  domain-a/
    types.strux          → module: domain-a/types
    pipelines/
      intake.strux       → module: domain-a/pipelines/intake
```

### Imports (`@use`)

A file imports definitions from another module with `@use`:

```
@use "domain-a/types" { User, Grant }
@use "domain-a/types" { * }
@use "domain-a/types" { User AS DomainUser }
```

Rules:

1. `@use` must appear at the top of the file, before any `@type`,
   `@panel`, or `@context` declaration.
2. Named imports (`{ User, Grant }`) import only the listed names.
3. Wildcard import (`{ * }`) imports all exported names. SHOULD be
   avoided in production code (risk of collision, harder to audit).
4. Alias (`{ User AS DomainUser }`) renames on import to resolve
   collisions.
5. Relative paths are resolved from the importing file's directory.
   Absolute paths are resolved from the project root.
6. Circular imports are a compile error (`E_CIRCULAR_IMPORT`).

### Exports (`@export`)

By default, all top-level definitions in a file are **internal** (visible
only within the same directory subtree). To make a definition available
for import:

```
@export @type User { id: string, email: string }
@export @type Grant = union { nlnet: NLnetGrant, eu: EUGrant }
```

Rules:

1. `@export` is a modifier on `@type` and `@panel` declarations.
2. Unexported definitions are usable within the same directory and its
   children (directory-scoped visibility), but not importable by
   `@use` from outside that subtree.
3. `strux.context` files are never imported via `@use` — they
   participate in the config cascade only (ADR-016).
4. A module that exports nothing is valid (side-effect-free internal
   module).

### Hub packages

External shared packages (published to the Strux hub, per ADR-012)
use a package identifier with version:

```
@use "hub:openstrux/compliance@0.6" { PseudonymizeAlgo, EncryptionKey }
@use "hub:acme/shared-types@1.2" { Customer, Order }
```

Rules:

1. Hub packages are declared in `snap.lock` with pinned version and
   content hash (ADR-014), ensuring deterministic builds.
2. The `hub:` prefix distinguishes external packages from local paths.
3. Hub packages follow semver. The version in `@use` is a minimum
   compatible version; `snap.lock` pins the exact resolved version.
4. Hub package types participate in the same type system — union
   narrowing, type paths, adapter resolution all work identically.

### Interaction with `strux.context`

The module system and context cascade are orthogonal:

- **`strux.context`** shares *config*: `@dp`, `@access`, named
  `@source`/`@target`, `@ops`, `@sec`. Inherited implicitly by
  directory position. No `@use` needed.
- **`@use`** shares *definitions*: `@type`, `@panel`, and (future)
  `@fn`. Imported explicitly by declaration. No cascading.

This separation ensures that config inheritance (which is positional
and implicit) does not interfere with definition imports (which are
explicit and auditable).

### Interaction with syntax-reference.md

The syntax-reference.md is the self-sufficient LLM entry point for
Strux. When the module system is implemented, a compact summary will
be added to syntax-reference.md (similar to the existing Context
Inheritance section), keeping the reference under its token budget.
The full module system spec will live in a dedicated spec file.

## Alternatives Considered

### Implicit global scope (status quo)

Every definition visible everywhere. Simple but does not scale.
Name collisions require manual convention (prefixing). No way to
publish or consume shared packages.

### Path-based auto-import (like Go)

Import based on directory structure without explicit `@use`. Lower
ceremony but harder to audit ("where does this type come from?") and
incompatible with the explicit-over-implicit design principle.

### Namespace blocks

```
@namespace acme.domain-a { @type User { ... } }
```

Adds a scoping construct but doesn't solve the import problem.
Namespaces may be added later as syntactic sugar over the file-based
module system.

## Consequences

- Every `.strux` file that uses types from another file will need
  `@use` declarations. This is more verbose than implicit global
  scope but makes dependencies explicit and auditable.
- The compiler must resolve imports before type checking — adds a
  module resolution phase to the pipeline (Parse → **Resolve** →
  Normalize → IR → Validate → Emit).
- Hub package support requires `snap.lock` integration (ADR-014)
  and hub resolution (ADR-012).
- Token cost of `@use` lines is small (~5-10 tokens per import)
  and amortized across the definitions used.

## Open Questions

1. **Re-export.** Should a module be able to re-export imported
   types? e.g., `@export @use "domain-a/types" { User }`. Useful
   for facade modules but adds complexity.
2. **Version conflicts.** If two hub packages depend on different
   versions of a shared type, how is this resolved? Fail at lock
   time (strict) or allow diamond dependencies with compatible
   semver (lenient)?
3. **Functions.** When user-defined functions are added to Strux,
   they will need to be exportable and importable under this same
   system. The `@export @fn` pattern should work naturally but
   needs validation.
4. **Panel imports.** Importing a panel from another module enables
   composition (panel-of-panels). This is a significant capability
   that may warrant its own ADR if pursued.

## References

- ADR-012: Hub Artifact Format and Governance
- ADR-014: Lock File Semantics
- ADR-016: Token-Efficient Authoring Pattern
- `specs/core/config-inheritance.md` — context cascade (orthogonal)
- `specs/core/syntax-reference.md` — LLM entry point (will add module summary)
