# ADR-019: Generated Output as Native Package

- **Status:** Proposed
- **Date:** 2026-03-21
- **Updated:** 2026-03-21
- **Context:** The current TypeScript target adapter emits generated files
  directly into the user's project tree (types/, lib/schemas/, app/api/,
  prisma/). These files carry a `// do not edit` header but nothing enforces
  that boundary. Developers can accidentally modify generated code, and
  regeneration risks overwriting hand-written changes when generated and
  hand-written files share directories. Additionally, rod implementations
  are emitted as stubs with TODO comments, requiring hand-editing — which
  contradicts the structure-first principle. If the `.strux` source fully
  specifies the pipeline (receive → validate → store → respond), the
  generator has all information needed to emit complete implementations.

## Decision

### 1. Fully generated rod implementations (no stubs)

Rod implementations MUST be fully generated from the `.strux` pipeline
definition. If a rod's behavior is fully specified by its type, position in
the panel graph, and connected rods, the emitted code MUST be a complete,
working implementation — not a stub with TODO comments.

A `receive` rod connected to `validate → write-data → respond` emits a
complete handler that parses input, validates with the schema, writes to
the database, and returns the response. No hand-editing required.

This is the prerequisite for packaging: if all generated code is purely
derived, none of it needs hand-editing, and the entire output can live in
an immutable package.

### 2. Generated output as a package-shaped build artifact

`strux build` emits a self-contained, importable **package-shaped
directory** in the target ecosystem's native format. "Package-shaped" means
the output has its own metadata, barrel exports, and stable import surface
— but it lives in a **repo-owned directory**, not inside a package
manager's install tree.

The key distinction: **logical package identity is separate from physical
install location.** The output is structured like a package (stable imports,
metadata, full replacement semantics) without being managed by the package
manager.

Each target adapter defines a **packaging profile**:

| Concern | TypeScript (npm) | Python (pip) | Go | Rust (cargo) |
|---|---|---|---|---|
| Package format | `package.json` | `pyproject.toml` | `go.mod` | `Cargo.toml` |
| Default output | `.openstrux/build/` | `.openstrux/build/` | `internal/struxgen/` | `.openstrux/build/` |
| Import name | `@openstrux/build` | `openstrux_build` | `project/internal/struxgen` | `openstrux_build` |
| Import mechanism | `tsconfig.json` paths | editable install / sys.path | Go module path | `Cargo.toml` path dep |

The CLI orchestrates; the adapter owns the packaging:

```
interface TargetAdapter {
  emit(ir: IR): GeneratedFile[]
  package(files: GeneratedFile[]): PackageOutput
}

interface PackageOutput {
  outputDir: string
  metadata: Record<string, any>   // package.json / pyproject.toml / etc.
  entrypoints: string[]           // barrel exports / __init__.py / mod.rs
}
```

#### Why repo-owned, not `node_modules`

Prisma ORM's evolution is instructive. Prisma historically generated its
client into `node_modules/.prisma/client`. Starting with Prisma 6.6 this
triggers a deprecation warning, and Prisma 7 requires a custom `output`
path. Prisma now recommends placing generated source inside the application
codebase because it is bundled and processed like source code.

The same reasoning applies to OpenStrux output:

- Generated code is **application-specific source**, not a reusable
  third-party dependency
- Package managers may prune, hoist, or deduplicate `node_modules`
  unpredictably
- CI pipelines, containers, and reproducible builds often avoid
  install-time code generation
- Editor caches and TypeScript project references work more reliably with
  repo-owned paths
- `node_modules` content disappears on `npm install` / `rm -rf node_modules`

#### How stable imports work without `node_modules`

For TypeScript, `strux init` configures `tsconfig.json` path aliases:

```json
{
  "compilerOptions": {
    "paths": {
      "@openstrux/build": [".openstrux/build"],
      "@openstrux/build/*": [".openstrux/build/*"]
    }
  }
}
```

For bundlers that don't read tsconfig paths (e.g., Webpack without
`tsconfig-paths-webpack-plugin`), `strux init` also configures the
bundler's alias (e.g., `resolve.alias` in Webpack, automatic in Next.js).

The import surface is identical regardless of physical location:

```typescript
import { Proposal } from "@openstrux/build"
import { ProposalSchema } from "@openstrux/build/schemas"
```

### 3. Config-driven target selection

The target stack is configured in `strux.config.yaml` using npm-style
package names and semver ranges for dependency versioning:

```yaml
target:
  base: typescript@~5.5
  framework: next@^15.0
  orm: prisma@^6.0
  validation: zod@^3.23
  runtime: node@>=20
```

Each dependency range resolves to a certified adapter from the hub. The
resolved versions are pinned in `snap.lock`:

```yaml
adapters:
  typescript: 5.5.4
  next: 15.1.2
  prisma: 6.1.0
  zod: 3.23.8
  adapter/nextjs@15: 1.2.0
  adapter/prisma@6: 1.0.0
  adapter/zod@3: 1.1.0
```

This naming applies uniformly: `next@^15.0` and `nestjs@^10.0` are
different framework targets, not variations of a single "TypeScript"
target. The framework version is a first-class concern because different
major versions may have fundamentally different architectures (e.g.,
Next.js 15 has middleware, Next.js 16 does not).

### 4. Adapter management

**Official adapters** live in a monorepo with shared CI and cross-adapter
integration tests:

```
openstrux-adapters/
  base/ts/              # types, interfaces — shared by all TS stacks
  validation/zod/       # Zod schemas
  orm/prisma/           # Prisma schema + queries
  framework/nextjs/     # Next.js route handlers
  framework/nestjs/     # NestJS controllers, services, modules
```

**Community adapters** live in their own repos and are published to the
hub with a certification level:

| Level | Meaning |
|---|---|
| `certified` | Official, tested by core team |
| `verified` | Community-built, passes conformance suite |
| `community` | Community-built, self-attested |

Each adapter declares its compatibility range in its manifest, using the
same semver range syntax as `strux.config.yaml`:

```yaml
name: adapter/nextjs
version: 1.2.0
supports:
  framework: next@>=14.0 <17.0
  base: typescript@>=5.0
  orm:
    - prisma@>=5.0 <7.0
    - drizzle@>=0.30
  validation:
    - zod@>=3.20
  runtime:
    - node@>=18
    - bun@>=1.0
```

### 5. Onboarding: `strux init`

The onboarding flow detects the existing project stack, configures import
aliases, and builds immediately:

```
$ npm install @openstrux/cli
$ strux init

  Detected: next@15.1.2, prisma@6.1.0, zod@3.23.8, typescript@5.5.4

  ? Use detected stack? (Y/n) Y
  ✓ Wrote strux.config.yaml
  ✓ Configured tsconfig.json paths for @openstrux/build
  ✓ Added .openstrux/ to .gitignore
  ✓ Wrote src/strux/starter.strux
  ✓ Built @openstrux/build → .openstrux/build/ (4 files)

  Try: import { HealthCheck } from "@openstrux/build"
```

`strux init` creates a minimal starter `.strux` file and immediately runs
`strux build`, so imports resolve and IDE autocomplete works from the first
moment. For subsequent setups (cloning the repo, CI), `strux build` is run
explicitly as a build step — no implicit install hooks.

### 6. Discoverability

Not every framework × ORM × validation combination has an adapter. The
CLI provides discovery commands:

- `strux init` — interactive setup, shows only supported combinations
- `strux doctor` — validates config against available adapters, reports
  mismatches with suggestions
- `strux adapters list` — browse all available adapters with version ranges
- `strux adapters check <combo>` — test whether a specific combination
  resolves

### 7. Regeneration semantics

- `strux build` replaces the entire output directory contents
- The output is a derived artifact — it MUST NOT contain hand-written code
- The output directory SHOULD be `.gitignore`d by default (generated on
  build, not committed) or committed as a build artifact depending on team
  policy
- Determinism guarantee (ADR-000) still holds: same source + same lock =
  identical output
- `strux build` is an explicit command, not an implicit install hook. CI
  pipelines and containers run it as a build step, which is predictable
  and auditable

### 8. Consumer integration

The consumer project owns application entry points, custom business logic,
configuration, and tests. It imports from the generated package:

```typescript
import { Proposal } from "@openstrux/build"
import { ProposalSchema } from "@openstrux/build/schemas"
import { intakeProposals } from "@openstrux/build/handlers"
```

Import paths are stable across regenerations. The same paths work
regardless of the configured stack — only the implementation behind them
changes.

The generated output directory structure:

```
.openstrux/build/
  package.json          # metadata (name, exports map, types)
  tsconfig.json         # strict, composite, declaration
  index.ts              # barrel re-exports
  types/
    Proposal.ts
    ReviewStatus.ts
  schemas/
    Proposal.schema.ts
    index.ts
  handlers/
    intake-proposals.ts
    index.ts
  guards/
    intake.guard.ts
  prisma/
    schema.prisma
```

## Alternatives Considered

**Loose files with `// do not edit` header (current approach):** Simple but
fragile. No structural enforcement of the generated/hand-written boundary.
Regeneration is risky when generated and hand-written files share
directories. Superseded.

**`node_modules` injection (Prisma legacy pattern):** Zero config, familiar.
But Prisma itself is moving away from this: Prisma 7 requires a custom
output path and recommends keeping generated code in the application tree.
Generated application-specific code is source, not a dependency. Package
managers may prune, hoist, or deduplicate unpredictably. CI reproducibility
suffers when install-time codegen is required. Rejected as default;
available as an opt-in adapter profile for teams that prefer it.

**Workspace package (monorepo-style):** Requires workspace tooling
configuration (pnpm/yarn workspaces). Adds setup friction and only works
in monorepo-structured projects. Rejected as default; teams may choose
this layout if preferred.

**npm-published package (registry-based):** Adds a publish step and
versioning overhead. Generated code is project-specific and changes with
every `.strux` edit — registry publishing adds friction without value.
Rejected for the default workflow.

**Target named `target-ts` (language-level):** TypeScript is the language,
not the stack. A team using Express + Drizzle produces fundamentally
different code from one using Next.js + Prisma. Naming the target after
the framework is honest about what the adapter produces. Rejected.

**Stubs with TODO comments (current approach):** If the `.strux` source
specifies the full pipeline, the generator has all information needed to
emit complete code. Stubs push implementation responsibility back to the
developer, defeating the purpose of a structural language. Superseded.

## Consequences

- Generated code is structurally isolated — a dedicated directory with
  full replacement semantics prevents accidental edits
- The output is "package-shaped without being package-manager-owned" —
  stable imports via path aliases, adapter-owned metadata, but repo-owned
  physical location
- Rod implementations are complete and working, not stubs — the generated
  package is immediately usable
- Each target adapter owns both code emission and packaging for its
  ecosystem, keeping the CLI target-agnostic
- Framework version is a first-class concern — different major versions
  may require entirely different adapters
- Config uses standard npm semver ranges, so the versioning model is
  immediately familiar to developers
- The adapter ecosystem supports both official (monorepo, certified) and
  community (independent, verified/self-attested) contributions
- `strux init` configures path aliases and runs an initial build —
  imports work from the first moment with no red squiggles
- Teams can discover supported combinations before committing to a stack
- `strux build` is an explicit build step, not an implicit install hook —
  predictable in CI, containers, and reproducible builds
- The build command (`strux build`) replaces the current generation
  command, reflecting that the output is a complete artifact
