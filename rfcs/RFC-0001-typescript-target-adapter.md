# RFC-0001: TypeScript Target Adapter Contract

- **Status:** Accepted
- **Proposed by:** Olivier Fabre
- **Date:** 2026-03-20
- **Updated:** 2026-03-21 (ADR-019: emit/package split, framework-specific naming)

---

## Summary

This RFC defines the normative contract for OpenStrux target adapters, using the Next.js adapter
as the reference implementation. An adapter receives a validated AST, manifest, and resolved
options, and produces two outputs: `emit()` returns generated source files as an array of
`GeneratedFile` objects; `package()` wraps those files into an ecosystem-native build artifact.
A static adapter registry maps framework names to adapter implementations. This contract is the
extension point for all future code-generation targets (Beam Python, Terraform, etc.).

## Motivation

The OpenStrux toolchain can parse and validate `.strux` sources but produces nothing runnable.
Target generation is the core value proposition: from a validated, manifest-stamped AST the
generator emits deployable artifacts. Without a stable adapter contract, every target would need
to wire directly into the engine, making the system non-extensible.

The Next.js target is needed immediately for the v0.6.0 demo (NLnet grant-workflow use case:
Next.js/Prisma/Zod/Keycloak). RFC-0001 stabilises the contract first so that the Next.js adapter
is not an accidental template — it is an intentional reference implementation of a specified
interface.

## Detailed Design

### Adapter Contract

```typescript
/**
 * A single generated output file.
 */
export interface GeneratedFile {
  /** Relative path from the package output directory. */
  path: string;
  /** File content as a string. */
  content: string;
  /** Language identifier (e.g., "typescript", "prisma", "json"). */
  lang: string;
}

/**
 * Resolved dependency entry from config + adapter manifest matching.
 */
export interface ResolvedDep {
  name: string;
  version: string;
  adapter: string;
}

/**
 * Options passed to the adapter after config resolution.
 */
export interface ResolvedOptions {
  framework: ResolvedDep;
  orm: ResolvedDep;
  validation: ResolvedDep;
  runtime: ResolvedDep;
}

/**
 * The output of the adapter's package() method.
 */
export interface PackageOutput {
  /** Output directory relative to the project root (default: ".openstrux/build"). */
  outputDir: string;
  /** Ecosystem metadata files: package.json, tsconfig.json, etc. */
  metadata: GeneratedFile[];
  /** Barrel export files: index.ts, schemas/index.ts, handlers/index.ts. */
  entrypoints: GeneratedFile[];
}

/**
 * The adapter interface every target must implement.
 */
export interface Adapter {
  /** Adapter name, matching the registry key (e.g. "nextjs"). */
  name: string;

  /**
   * Emit source files from a validated AST and manifest.
   * All GeneratedFile.path values are relative to the package output directory.
   *
   * @param ast      - Validated OpenStrux AST (array of top-level nodes)
   * @param manifest - Parsed mf.strux.json manifest object
   * @param options  - Resolved adapter options (framework, orm, validation, runtime)
   * @returns Array of generated source files. Order is not normative.
   */
  emit(
    ast: TopLevelNode[],
    manifest: Manifest,
    options: ResolvedOptions,
  ): GeneratedFile[];

  /**
   * Wrap emitted files in an ecosystem-native package.
   *
   * @param files - The files returned by emit()
   * @returns Package metadata and entrypoints
   */
  package(files: GeneratedFile[]): PackageOutput;
}
```

### Top-Level Node Union

```typescript
import type { TypeRecord, TypeEnum, TypeUnion } from "@openstrux/ast";
import type { Panel } from "@openstrux/ast";

export type TopLevelNode = TypeRecord | TypeEnum | TypeUnion | Panel;
```

### Manifest Type

```typescript
export type Manifest = Record<string, unknown>;
```

### Adapter Registry

The registry maps framework names to adapter implementations.

```typescript
const registry: Map<string, Adapter> = new Map();

export function registerAdapter(framework: string, adapter: Adapter): void {
  registry.set(framework, adapter);
}

export function getAdapter(framework: string): Adapter {
  const adapter = registry.get(framework);
  if (!adapter) throw new UnknownTargetError(framework);
  return adapter;
}

export function listTargets(): string[] {
  return Array.from(registry.keys());
}
```

### UnknownTargetError

```typescript
export class UnknownTargetError extends Error {
  constructor(framework: string) {
    super(`No adapter registered for target: "${framework}"`);
    this.name = "UnknownTargetError";
  }
}
```

### Top-Level `generate()` Function

```typescript
export function generate(
  ast: TopLevelNode[],
  manifest: Manifest,
  options: ResolvedOptions,
): GeneratedFile[] {
  const adapter = getAdapter(options.framework.name);
  return adapter.emit(ast, manifest, options);
}
```

### Framework-Specific Adapter Keys

Adapters are registered under their framework name, not a language name. For v0.6.0 the only
registered key is `"nextjs"`.

```typescript
import { NextJsAdapter } from "./adapters/nextjs/index.js";
registerAdapter("nextjs", NextJsAdapter);
```

The registry is keyed on strings so future targets (`"nestjs"`, `"beam-python"`, `"terraform"`)
can be registered without changes to the engine.

### Adapter Lifecycle

Adapters are stateless: they receive inputs, return outputs, and do not mutate shared state.
`emit()` MAY accumulate state within a single call (e.g., Prisma schema accumulator) but MUST
NOT retain state across calls.

### Compatibility Manifest

Each adapter declares the framework and library versions it supports:

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

The generator resolves the project's `strux.config.yaml` semver ranges against these manifests
to produce `ResolvedOptions`.

### Output File Conventions

- All paths in `GeneratedFile.path` are relative to the package output directory
- The output directory defaults to `.openstrux/build/` and is determined by `PackageOutput.outputDir`
- The generator does NOT write files to disk — that is the CLI's responsibility
- Files with the same path returned by the adapter are not accumulated; the adapter uses internal
  state to combine multiple contributions to the same file (e.g., `prisma/schema.prisma`)

### Generator Specification Reference

The normative engine spec is in `specs/generator/generator.md`. The normative Next.js adapter
mapping rules are in `specs/modules/target-nextjs/generator.md` and `specs/modules/target-nextjs/rods.md`.

## Drawbacks

- Static registry requires recompilation to add a new target — acceptable for v0.6.0.
- Manifest passed as `Record<string, unknown>` loses type safety — mitigated by the typed
  `@openstrux/manifest` package that callers should use before invoking the generator.

## Alternatives Considered

**Single `generate()` method on the adapter (no `package()`):** The original RFC-0001 design.
Deferred packaging logic to the caller. Does not support ecosystem-native packaging profiles
(package.json, barrel exports) without ad-hoc conventions. Superseded by ADR-019.

**Plugin discovery via `package.json` exports:** Deferred. Adds significant complexity with no
benefit for the single-target v0.6.0 milestone.

**Single monolithic generator function (no adapter abstraction):** Rejected. The spec already
references multiple targets (TypeScript, Beam Python). Coupling the engine to a single target
would require breaking changes to add the second target.

**`GeneratedFile` with `Buffer` content instead of `string`:** Rejected. All OpenStrux generated
artifacts are text files. `string` is simpler and serialisable without encoding concerns.

**Adapter registered as `"typescript"` (language-level):** Rejected. TypeScript is the language,
not the stack. A team using Express + Drizzle produces fundamentally different code from one
using Next.js + Prisma. Framework-level naming is honest. See ADR-019.

## Unresolved Questions

- Should the adapter contract include a `validate()` method for pre-generation checks? Deferred.
- Should `GeneratedFile` include a content hash for cache invalidation? Deferred to post-0.6.0.

## Implementation Plan

1. `openstrux-spec`: This RFC + `specs/generator/generator.md` + `specs/modules/target-nextjs/` + golden fixtures
2. `openstrux-core`: Update `packages/generator/` implementing the contract defined here

## Annex A: Canonical Source Form for Content Hashing

This annex defines the exact canonicalization algorithm used to produce a stable content hash for
any generated file.

### Algorithm

Given a `GeneratedFile` with `content: string`:

1. **Normalize line endings:** Replace all `\r\n` and lone `\r` with `\n`.
2. **Strip trailing whitespace:** For each line, remove all trailing space/tab characters.
3. **Strip leading blank lines:** Remove all leading blank lines.
4. **Strip trailing blank lines:** Remove all trailing blank lines.
5. **Append final newline:** Ensure the content ends with exactly one `\n`.
6. **Encode:** Convert the resulting string to UTF-8 bytes.
7. **Hash:** Compute SHA-256 of the UTF-8 bytes.
8. **Format:** Hex-encode the 32-byte digest (lowercase, no prefix).

### Reference Implementation (TypeScript)

```typescript
import { createHash } from "node:crypto";

export function canonicalHash(content: string): string {
  let s = content.replace(/\r\n/g, "\n").replace(/\r/g, "\n");
  s = s.split("\n").map((l) => l.trimEnd()).join("\n");
  s = s.replace(/^\n+/, "").replace(/\n+$/, "");
  s = s + "\n";
  return createHash("sha256").update(s, "utf8").digest("hex");
}
```

### Test Vectors

| Input (escaped)   | Expected SHA-256 hex                                               |
| ----------------- | ------------------------------------------------------------------ |
| `"hello\n"`       | `5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03` |
| `"hello\r\n"`     | `5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03` |
| `"  hello  \n"`   | `2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824` |
| `"\n\nhello\n\n"` | `5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03` |

---

## Decision Record

| Date       | Event                                                              |
| ---------- | ------------------------------------------------------------------ |
| 2026-03-20 | Draft opened — Olivier Fabre (AI-assisted)                         |
| 2026-03-20 | Self-accepted by Olivier Fabre (bootstrap phase, sole maintainer)  |
| 2026-03-21 | Updated for ADR-019: emit/package split, nextjs adapter name       |
