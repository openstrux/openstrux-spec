# Standard Rods

Standard rods are certified composite rods that ship with core. They differ from basic rods
(atomic, 18 total) and hub rods (community-contributed) in these ways:

| Property | Basic | Standard | Hub |
|---|---|---|---|
| Defined in | `specs/modules/rods/` | `specs/modules/rods/standard/` | Hub registry |
| Implemented in | `packages/generator/src/adapters/*/rods/` | `packages/generator/src/adapters/*/rods/standard/` | Hub package |
| Certification | Project (each rod) | Project (composition as a unit) | Community (trust varies) |
| Composition | Atomic primitive | Expands to basic rod sub-graph | Arbitrary |
| Author override | Not possible | Not possible | Allowed (via config) |

## Expansion model

During IR lowering, each standard rod is replaced by a deterministic sub-graph of basic rods.
The sub-graph depends on the rod's configuration at compile time. The expansion is recorded in
the lock file so that the same source always produces the same sub-graph.

```
standard-rod { cfg }
    ↓  (IR lowering)
basic-rod-A → basic-rod-B → basic-rod-C
```

The `@cert` block on a standard rod certifies the **composition** — the expansion wiring and
the semantics of the resulting sub-graph — not each individual basic rod separately.

## Certification

Standard rod certification works identically to basic rod certification (see
[`specs/core/conformance.md`](../../core/conformance.md)), except the scope is the composition:

```
@cert {
  tested: {
    framework: "gdpr",
    fields: [...],
    expansion: "validate → pseudonymize → guard"
  }
  hash: "sha256:..."
  version: "0.6.0"
}
```

The hash covers the expansion config, not the source text.

## Standard rods in this version

| Rod | Category | Status |
|---|---|---|
| `private-data` | Privacy | Normative (v0.6) |

## Adding a new standard rod

New standard rods require:

1. A spec definition in this directory (`specs/modules/rods/standard/<name>.strux`)
2. A normative entry in `specs/modules/rods/overview.md`
3. Expansion rules in `specs/core/semantics.md`
4. Valid and invalid conformance fixtures
5. Core implementation in `packages/generator/src/adapters/*/rods/standard/`
6. Manifesto benchmark coverage (see `MANIFESTO_BENCHMARKS.md`)
