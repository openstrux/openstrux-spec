# RFC-0001: TypeScript Target Adapter Contract

- **Status:** Accepted
- **Proposed by:** Olivier Fabre
- **Date:** 2026-03-20

---

## Summary

This RFC defines the normative contract for OpenStrux target adapters, using the TypeScript adapter
as the reference implementation. An adapter receives a validated AST and manifest, and returns an
array of `GeneratedFile` objects. A static adapter registry maps target names to adapter
implementations. This contract is the extension point for all future code-generation targets
(Beam Python, Terraform, etc.).

## Motivation

The OpenStrux toolchain can parse and validate `.strux` sources but produces nothing runnable.
Target generation is the core value proposition: from a validated, manifest-stamped AST the
generator emits deployable artifacts. Without a stable adapter contract, every target would need
to wire directly into the engine, making the system non-extensible.

The TypeScript target is needed immediately for the v0.6.0 demo (NLnet grant-workflow use case:
Next.js/Prisma/Zod/Keycloak). RFC-0001 stabilises the contract first so that the TypeScript adapter
is not an accidental template — it is an intentional reference implementation of a specified
interface.

## Detailed Design

### Adapter Contract

```typescript
/**
 * A single generated output file.
 */
export interface GeneratedFile {
  /** Relative path from the generator output root. */
  path: string;
  /** File content as a string. */
  content: string;
  /** Language identifier (e.g., "typescript", "prisma", "markdown"). */
  lang: string;
}

/**
 * Options passed to the generator and adapter.
 */
export interface GenerateOptions {
  /** Target identifier. Must match a registered adapter key. */
  target: string;
  /** Optional: Next.js version string. Defaults to "14". */
  nextVersion?: string;
  /** Optional: additional target-specific options. */
  [key: string]: unknown;
}

/**
 * The adapter interface every target must implement.
 */
export interface Adapter {
  /**
   * Generate output files from a validated AST and manifest.
   *
   * @param ast     - Validated OpenStrux AST (array of top-level nodes)
   * @param manifest - Parsed mf.strux.json manifest object
   * @param options  - Generator options (target, nextVersion, etc.)
   * @returns Array of generated files. Order within the array is not normative.
   */
  generate(
    ast: TopLevelNode[],
    manifest: Manifest,
    options: GenerateOptions,
  ): GeneratedFile[];
}
```

### Top-Level Node Union

```typescript
import type { TypeRecord, TypeEnum, TypeUnion } from "@openstrux/ast";
import type { Panel } from "@openstrux/ast";

export type TopLevelNode = TypeRecord | TypeEnum | TypeUnion | Panel;
```

### Manifest Type

The manifest is the parsed `mf.strux.json` object. For this RFC the adapter receives it as
`Record<string, unknown>`. A typed `Manifest` interface will be defined in `@openstrux/manifest`
(existing package); adapters should treat it as opaque unless they need specific fields.

```typescript
export type Manifest = Record<string, unknown>;
```

### Adapter Registry

The registry maps string target names to adapter implementations.

```typescript
const registry: Map<string, Adapter> = new Map();

export function registerAdapter(target: string, adapter: Adapter): void {
  registry.set(target, adapter);
}

export function getAdapter(target: string): Adapter {
  const adapter = registry.get(target);
  if (!adapter) throw new UnknownTargetError(target);
  return adapter;
}
```

### UnknownTargetError

```typescript
export class UnknownTargetError extends Error {
  constructor(target: string) {
    super(`No adapter registered for target: "${target}"`);
    this.name = "UnknownTargetError";
  }
}
```

### Top-Level `generate()` Function

```typescript
export function generate(
  ast: TopLevelNode[],
  manifest: Manifest,
  options: GenerateOptions,
): GeneratedFile[] {
  const adapter = getAdapter(options.target);
  return adapter.generate(ast, manifest, options);
}
```

### Target Selection

The caller sets `options.target` to the registered adapter key. For v0.6.0 the only registered key
is `"typescript"`. The registry is intentionally keyed on strings so that future targets (e.g.,
`"beam-python"`, `"terraform"`) can be registered at startup without changes to the engine.

### Adapter Lifecycle

Adapters are stateless pure functions: they receive inputs, return outputs, and do not mutate
shared state. The registry is populated at module load time (import side-effects). Adapter
implementations MAY accumulate state within a single `generate()` call (e.g., a Prisma schema
accumulator) but MUST NOT retain state across calls.

### Static Registry (v0.6.0)

For v0.6.0 the registry is hardcoded:

```typescript
import { TypeScriptAdapter } from "./adapters/typescript/index.js";
registerAdapter("typescript", TypeScriptAdapter);
```

Plugin discovery (npm package scanning, adapter manifests) is deferred to post-0.6.0.

### Output File Conventions

- All paths in `GeneratedFile.path` are relative to the generator output root.
- The output root is determined by the caller (e.g., a CLI flag `--out`).
- The generator does NOT write files to disk — that is the CLI's responsibility.
- Files returned by the adapter MAY be accumulated (multiple rods contributing to the same path);
  the engine accumulates by path before returning. If two adapter calls return the same path, the
  second content replaces the first unless the adapter uses internal accumulation.

### Generator Specification Reference

The normative mapping rules from AST nodes to TypeScript artifacts are defined in
`specs/modules/target-ts/generator.md`. That document is the source of truth for the TypeScript
adapter implementation and golden fixture validation.

## Drawbacks

- Static registry requires recompilation to add a new target — acceptable for v0.6.0.
- Manifest passed as `Record<string, unknown>` loses type safety — mitigated by the typed
  `@openstrux/manifest` package that callers should use before invoking the generator.

## Alternatives Considered

**Plugin discovery via `package.json` exports:** Deferred. Adds significant complexity with no
benefit for the single-target v0.6.0 milestone.

**Single monolithic generator function (no adapter abstraction):** Rejected. The spec already
references multiple targets (TypeScript, Beam Python). Coupling the engine to a single target
would require breaking changes to add the second target.

**`GeneratedFile` with `Buffer` content instead of `string`:** Rejected. All OpenStrux generated
artifacts are text files. `string` is simpler and serialisable without encoding concerns.

## Unresolved Questions

- Should the adapter contract include a `validate()` method for pre-generation checks? Deferred.
- Should `GeneratedFile` include a content hash for cache invalidation? Deferred to post-0.6.0.

## Implementation Plan

1. `openstrux-spec`: This RFC + `specs/modules/target-ts/generator.md` + golden fixtures
2. `openstrux-core`: New `packages/generator/` implementing the contract defined here

## Annex A: Canonical Source Form for Content Hashing

This annex defines the exact canonicalization algorithm used to produce a stable content hash for
any generated file. The hash is used by the manifest system to certify generated artifacts.

### Algorithm

Given a `GeneratedFile` with `content: string`:

1. **Normalize line endings:** Replace all `\r\n` and lone `\r` with `\n`.
2. **Strip trailing whitespace:** For each line, remove all trailing space/tab characters.
3. **Strip leading blank lines:** Remove all leading blank lines (lines containing only `\n`).
4. **Strip trailing blank lines:** Remove all trailing blank lines.
5. **Append final newline:** Ensure the content ends with exactly one `\n`.
6. **Encode:** Convert the resulting string to UTF-8 bytes.
7. **Hash:** Compute SHA-256 of the UTF-8 bytes.
8. **Format:** Hex-encode the 32-byte digest (lowercase, no prefix).

### Reference Implementation (TypeScript)

```typescript
import { createHash } from "node:crypto";

export function canonicalHash(content: string): string {
  // Step 1: normalize line endings
  let s = content.replace(/\r\n/g, "\n").replace(/\r/g, "\n");
  // Step 2: strip trailing whitespace per line
  s = s
    .split("\n")
    .map((l) => l.trimEnd())
    .join("\n");
  // Step 3-4: strip leading/trailing blank lines
  s = s.replace(/^\n+/, "").replace(/\n+$/, "");
  // Step 5: append final newline
  s = s + "\n";
  // Steps 6-8: SHA-256 hex
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

Note: `"hello\n"` after canonicalization is `"hello\n"`. SHA-256(`hello\n`) = the first vector.

---

## Decision Record

| Date       | Event                                                             |
| ---------- | ----------------------------------------------------------------- |
| 2026-03-20 | Draft opened — Olivier Fabre (AI-assisted)                        |
| 2026-03-20 | Self-accepted by Olivier Fabre (bootstrap phase, sole maintainer) |
