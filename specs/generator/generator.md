# OpenStrux — Generator Engine Specification

**Status:** Normative
**Spec version:** 0.6.0
**Implements:** RFC-0001 (TypeScript Target Adapter) — extended by ADR-019

---

## 1. Overview

The generator engine accepts a validated OpenStrux AST and a resolved adapter set, then
orchestrates the full build pipeline: emit source files and package them into an ecosystem-native
build artifact. Target adapters are registered by framework name; the engine dispatches to the
correct adapter based on config-resolved options.

---

## 2. Adapter Interface

Every target adapter MUST implement both methods:

```typescript
interface Adapter {
  name: string

  /**
   * Emit source files from a validated AST and manifest.
   * Returns package-relative paths (not project-root paths).
   */
  emit(
    ast: TopLevelNode[],
    manifest: Manifest,
    options: ResolvedOptions,
  ): GeneratedFile[]

  /**
   * Wrap emitted files in an ecosystem-native package.
   * Returns the output directory, metadata files, and barrel entrypoints.
   */
  package(files: GeneratedFile[]): PackageOutput
}
```

### GeneratedFile

```typescript
interface GeneratedFile {
  /** Path relative to the package output directory (not the project root). */
  path: string
  /** File content as a string. */
  content: string
  /** Language identifier: "typescript", "prisma", "json", etc. */
  lang: string
}
```

### PackageOutput

```typescript
interface PackageOutput {
  /** Output directory relative to the project root. Default: ".openstrux/build". */
  outputDir: string
  /** Ecosystem metadata files: package.json, tsconfig.json, etc. */
  metadata: GeneratedFile[]
  /** Barrel export files: index.ts, schemas/index.ts, handlers/index.ts, etc. */
  entrypoints: GeneratedFile[]
}
```

### ResolvedOptions

```typescript
interface ResolvedOptions {
  framework: ResolvedDep   // e.g. { name: "next", version: "15.1.2", adapter: "adapter/nextjs@1.2.0" }
  orm: ResolvedDep
  validation: ResolvedDep
  runtime: ResolvedDep
}

interface ResolvedDep {
  name: string
  version: string
  adapter: string
}
```

---

## 3. Adapter Registry

The registry maps framework names to adapter implementations.

```typescript
const registry: Map<string, Adapter> = new Map()

function registerAdapter(framework: string, adapter: Adapter): void
function getAdapter(framework: string): Adapter   // throws UnknownTargetError if not found
function listTargets(): string[]
```

### Scenario: Unknown framework throws

- **WHEN** `getAdapter("no-such-framework")` is called
- **THEN** `UnknownTargetError` SHALL be thrown with the framework name in the message

---

## 4. Generator Pipeline

The `strux build` command orchestrates the following steps in order:

1. **Read config** — parse `strux.config.yaml` at the project root
2. **Resolve adapters** — match config semver ranges against bundled adapter manifests; produce `ResolvedOptions`
3. **Parse** — parse all `.strux` files in the project → AST
4. **Validate** — type-check and scope-validate the AST
5. **Emit** — call `adapter.emit(ast, manifest, resolvedOptions)` → `GeneratedFile[]`
6. **Package** — call `adapter.package(files)` → `PackageOutput`
7. **Write** — write all files to `packageOutput.outputDir` (default `.openstrux/build/`)

### Scenario: Build pipeline runs end to end

- **WHEN** `strux build` is invoked with a valid config, valid `.strux` files, and a compatible adapter
- **THEN** the output directory SHALL contain all emitted source files, metadata files, and barrel entrypoints

---

## 5. Config Format

The project config lives at `strux.config.yaml` in the project root:

```yaml
target:
  base: typescript@~5.5
  framework: next@^15.0
  orm: prisma@^6.0
  validation: zod@^3.23
  runtime: node@>=20
```

All version strings MUST be npm-style semver ranges. The `framework` field determines which
adapter is selected. If no adapter satisfies all ranges, the build fails with a resolution error.

### Scenario: Config resolution produces ResolvedOptions

- **WHEN** config contains `framework: next@^15.0` and adapter `adapter/nextjs@1.2.0` declares `next@>=14.0 <17.0`
- **THEN** `ResolvedOptions.framework` SHALL be `{ name: "next", version: "15.1.2", adapter: "adapter/nextjs@1.2.0" }`

---

## 6. Adapter Manifest

Each adapter declares its compatibility ranges:

```yaml
name: adapter/nextjs
version: 1.2.0
supports:
  framework: next@>=14.0 <17.0
  base: typescript@>=5.0
  orm:
    - prisma@>=5.0 <7.0
  validation:
    - zod@>=3.20
  runtime:
    - node@>=18
```

For v0.6, adapters are bundled with the CLI. Hub-based discovery is future work.

### Scenario: Incompatible adapter not selected

- **WHEN** config requests `framework: next@^16.0` and only `adapter/nextjs@1.2.0` supporting `next@>=14.0 <16.0` is available
- **THEN** adapter resolution SHALL fail with a clear error message

---

## 7. Output Path Convention

All `GeneratedFile.path` values returned by `adapter.emit()` SHALL be relative to the package
output directory — not the project root. The engine prepends the output directory when writing
files to disk.

### Scenario: Type file path is package-relative

- **WHEN** the adapter emits a type file for `@type Proposal`
- **THEN** `GeneratedFile.path` SHALL be `types/Proposal.ts`, not `.openstrux/build/types/Proposal.ts`

---

## 8. Conformance

Each adapter implementation is tested by running `adapter.emit()` against conformance fixture ASTs
and diffing the output against golden files in `conformance/golden/<adapter-name>/`.

Normalisation rules (applied before diff):

1. Collapse consecutive blank lines to a single blank line
2. Trim trailing whitespace per line
3. Sort import statements alphabetically within each contiguous import block
