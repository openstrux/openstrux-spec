# OpenStrux v0.4 — Lock Semantics

## What a Lock Is

A lock file (`snap.lock`) captures the complete resolution state of an
OpenStrux compilation. It is the mechanism that guarantees the
determinism requirement: **same source + same lock = same output**.

The lock records everything that the compiler resolved during
compilation that is not directly expressed in the source files:
dependency versions, content hashes, adapter selections, and policy
resolutions.

## Why Locks Exist

Without a lock, two compilations of the same source may produce
different output if:

- A rod version in the Hub was updated between compilations
- An adapter was patched to a new version
- A named policy resolved to a different definition
- A `strux.context` file was modified in an ancestor directory

The lock freezes these resolutions. A build from locked source is
reproducible indefinitely, regardless of changes to the Hub, context
files, or external policy stores.

## What `snap.lock` Contains

```
snap.lock
├── meta
│   ├── spec_version:     "0.4"
│   ├── lock_version:     1
│   ├── created:          ISO-8601 timestamp
│   └── source_hash:      content hash of all input .strux files
├── dependencies
│   ├── rods[]
│   │   ├── type:         rod type name
│   │   ├── version:      resolved version
│   │   └── hash:         content hash of rod definition
│   ├── adapters[]
│   │   ├── type_path:    union leaf type (e.g., "db.sql.postgres")
│   │   ├── target:       emission target (e.g., "beam-python")
│   │   ├── version:      resolved adapter version
│   │   └── hash:         content hash of adapter implementation
│   └── policies[]
│       ├── name:         policy name
│       ├── version:      resolved version
│       └── hash:         content hash of policy definition
├── context_chain[]
│   ├── path:             relative path to strux.context file
│   └── hash:             content hash
└── resolution
    ├── type_paths{}       map of type path → resolved concrete type + hash
    └── sources{}          map of named source → resolved config + hash
```

<!-- GAP: The exact serialization format (JSON, TOML, etc.) is not yet
     specified. The structure above is normative; the encoding is TBD. -->

## Lock Creation

A lock is generated when:

1. **First compilation** — no `snap.lock` exists. The compiler resolves
   all dependencies, records versions and hashes, and writes `snap.lock`.
2. **Explicit update** — `strux lock update` re-resolves all
   dependencies to their latest compatible versions and rewrites the lock.
3. **Dependency change** — when a source file adds or removes a
   dependency (new rod type, new adapter, new policy), the compiler
   updates the affected lock entries.

### Inputs That Determine the Lock

- Hub registry state (available rod/adapter/policy versions)
- `strux.context` cascade (all ancestor context files)
- Source `.strux` files (type definitions, panel definitions)
- Version constraints (if specified in source or context)

## Lock Consumption

When a lock exists, the compiler uses it to reproduce a previous build:

1. Read `snap.lock`.
2. For each dependency, resolve to the **exact version and hash**
   recorded in the lock (not the latest available).
3. Verify content hashes match. If a hash mismatch is detected:
   - **MUST** emit a diagnostic warning.
   - **MAY** refuse to compile (strict mode) or continue with warning
     (permissive mode).
4. Proceed with compilation using locked versions.

A conformant implementation MUST support lock consumption. Lock
creation MAY be deferred to a separate tool.

## Lock Invalidation

The lock becomes invalid (requires update) when:

1. **Source change** — a `.strux` file adds or removes a dependency
   that is not in the lock.
2. **Context change** — a `strux.context` file in the resolution chain
   is modified (hash mismatch).
3. **Explicit request** — user runs `strux lock update`.
4. **Hub artifact removal** — a locked version is no longer available
   in the registry.

The lock does NOT become invalid when:

- A new version of a dependency is published (the lock pins the old
  version).
- Source files are reformatted without semantic change (content hash
  is computed on the normalized AST, not the source text).

## Relationship: Lock and Manifest

| Artifact | Purpose | When produced | What it contains |
|----------|---------|--------------|-----------------|
| `snap.lock` | Reproducibility | At compilation time | Dependency versions, content hashes, resolution state |
| `mf.strux.json` | Deployment | At compilation time | Fully resolved config, no inheritance, no shorthand |

The manifest is derived FROM the locked compilation. The lock guarantees
that the same manifest can be reproduced. They are complementary:

- **Lock** answers: "How do I reproduce this build?"
- **Manifest** answers: "What does this build contain?"

## Conformance Requirement

A conformant implementation MUST produce identical output given
identical source files and identical `snap.lock`. Specifically:

1. The compiled manifest (`mf.strux.json`) MUST be byte-identical.
2. Emitted target code MUST be byte-identical.
3. If any locked dependency cannot be resolved (missing from registry,
   hash mismatch), the compiler MUST fail with a diagnostic — it MUST
   NOT silently substitute a different version.

## Scope

This section is normative. The lock mechanism is REQUIRED for any
conformant toolchain. Implementations without Hub integration MAY
implement a simplified lock covering only context chain hashes and
source hashes.
