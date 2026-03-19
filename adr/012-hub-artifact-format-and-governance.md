# ADR-012: Hub Artifact Format and Governance

- **Status:** Accepted
- **Date:** 2026-03-19
- **Context:** ADR-000 establishes the Brick Registry (Layer 1) as the
  shared repository for reusable components. ADR-004, ADR-009, and
  ADR-010 introduce three artifact kinds (rods, adapters, policies)
  that the hub must store, version, and govern. This ADR defines the
  artifact model, publication rules, and governance framework.

## Decision

The hub is a **content-addressable, append-only registry** of typed
artifacts. Every artifact has a manifest, a content hash, and a
lifecycle status. Publication requires structural validation. Governance
is namespace-scoped.

## 1. Artifact Kinds

| Kind | What it contains | Published by | Referenced by |
|---|---|---|---|
| `rod` | Rod type definition + core logic | Rod authors | Panels (`name = rod-type`) |
| `adapter` | I/O translation for a union leaf + target | Adapter authors | Resolver (via interface match) |
| `interface` | Versioned union leaf type contract | Type authors | Adapters (`implements`), rods (`adapter-compat`) |
| `policy` | Named guard predicate (`.strux`) | Policy authors | Panels (`policy("name")`), `@access` blocks |
| `type` | Reusable type definitions (record, enum, union) | Type authors | Any `.strux` file via import |

All artifact kinds share the same manifest envelope. Kind-specific
fields extend it.

## 2. Artifact Manifest

Every artifact has a `manifest.strux.json`:

```json
{
  "kind": "rod",
  "name": "read-data",
  "namespace": "openstrux",
  "version": "1.2.0",
  "hash": "sha256:a1b2c3...",
  "status": "certified",
  "published_at": "2026-03-19T10:00:00Z",

  "spec_version": "0.5",
  "license": "Apache-2.0",
  "description": "Universal data source reader",

  "dependencies": {
    "interfaces": {
      "DataSource": "^1.0"
    },
    "adapter-compat": {
      "PostgresConfig": "^1.0",
      "KafkaConfig": "^1.0"
    }
  },

  "certification": {
    "level": "tested",
    "scope": { "source": "db.sql.*" },
    "tested_at": "2026-03-15T08:00:00Z",
    "hash": "sha256:d4e5f6..."
  },

  "signatures": [
    {
      "signer": "openstrux-ci",
      "method": "ed25519",
      "value": "base64:..."
    }
  ]
}
```

### Required Fields (all kinds)

| Field | Type | Description |
|---|---|---|
| `kind` | string | Artifact kind |
| `name` | string | Artifact name (unique within namespace) |
| `namespace` | string | Owning namespace |
| `version` | semver | Artifact version |
| `hash` | string | Content hash (sha256 of artifact payload) |
| `status` | enum | Lifecycle status (see §4) |
| `published_at` | ISO 8601 | Publication timestamp |
| `spec_version` | string | OpenStrux spec version this targets |

### Kind-Specific Fields

**Rod:** `dependencies.interfaces`, `dependencies.adapter-compat`,
`knots` (cfg/arg/in/out/err knot definitions).

**Adapter:** `implements` (interface name + version), `target` (emission
target: `beam-python`, `typescript`, etc.), `map` (field mapping).

**Interface:** `type_def` (the union leaf type definition), `fields`
(field names, types, constraints).

**Policy:** `rule` (guard predicate AST or source), `owner`, `applies_to`
(optional: resource patterns this policy governs).

**Type:** `type_def` (record, enum, or union definition).

## 3. Content Addressing

Every artifact version is **content-addressed**:

- `hash` = SHA-256 of the artifact payload (source files, manifest
  excluding `hash` and `signatures`)
- Two publications with identical content produce identical hashes
- The hub rejects duplicate publication (same name + version + different
  hash = `E_HASH_CONFLICT`)
- The hub allows re-publication of identical content (idempotent)

Content addressing enables:

- Lock file integrity verification (lock hash vs hub hash)
- Offline builds (artifact cache keyed by hash)
- Tamper detection (hash mismatch = `E_ARTIFACT_TAMPERED`)

## 4. Lifecycle Status

```
draft → published → certified → deprecated → yanked
```

| Status | Meaning | Resolvable? |
|---|---|---|
| `draft` | Work in progress, not yet published | No |
| `published` | Available but not certified | Yes (with `W_UNCERTIFIED`) |
| `certified` | Passed certification for declared scope | Yes |
| `deprecated` | Superseded, still resolvable | Yes (with `W_DEPRECATED`) |
| `yanked` | Security issue or critical bug | No (`E_ARTIFACT_YANKED`) |

Transitions:

- `draft → published`: Structural validation passes (see §5)
- `published → certified`: Certification suite passes for declared scope
- `certified → deprecated`: Maintainer marks as superseded
- `any → yanked`: Security response; immediate, non-reversible

## 5. Publication Rules

The hub validates at publication time:

### Structural Validation (all kinds)

- Manifest has all required fields
- Version follows semver
- Content hash matches payload
- `spec_version` is a supported version
- No name collision within namespace + kind

### Kind-Specific Validation

**Rod:**
- Knot definitions are well-typed
- `adapter-compat` ranges reference published interface artifacts
- At least one certified adapter satisfies each `adapter-compat` range
  for at least one target, OR status is set to `non-resolvable`
  (ADR-009 §3)

**Adapter:**
- `implements` references a published interface artifact
- Structural compatibility: adapter consumes all required fields
  defined by the interface
- `target` is a recognized emission target

**Policy:**
- `rule` parses as valid guard predicate (portable policy grammar)
- No circular policy references

**Interface:**
- `type_def` is a valid type definition
- Major version bump: breaking change detected vs previous major

## 6. Namespace Governance

Namespaces are the governance boundary:

| Namespace | Who controls | Examples |
|---|---|---|
| `openstrux` | Core team | Basic rods, standard interfaces |
| `openstrux-contrib` | Community (reviewed) | Community adapters, policies |
| `{org}` | Organization | `acme/read-crm`, `acme/gdpr-policy` |
| `{user}` | Individual | Personal experiments |

### Publication Permissions

- `openstrux` namespace: core team approval required
- `openstrux-contrib`: PR-based review, at least one core team approval
- Organization namespaces: governed by org admin
- User namespaces: self-service

### Certification Authority

- Self-certified: author runs test suite, declares certification
- Peer-certified: another party runs independent test suite
- Hub-certified: hub CI runs conformance suite against published
  artifact

The certification level is recorded in the manifest. Consumers can
set resolver policy (ADR-009 §2) to require specific certification
authorities.

## 7. Versioning Contract

All artifact kinds follow semver:

| Change type | Version bump | Consumer impact |
|---|---|---|
| Bug fix, no interface change | Patch | Compatible |
| New optional capability | Minor | Compatible |
| Breaking interface change | Major | Requires consumer update |

For **interfaces** specifically (ADR-009 §1):

- New optional field → minor
- New required field, removed field, type change → major
- Field constraint relaxation → minor
- Field constraint tightening → major

## 8. Conformance Cases

### Valid

- `v012-rod-manifest.json` — Well-formed rod manifest with all required
  fields. Publication succeeds.
- `v012-adapter-compat.json` — Rod with `adapter-compat` range satisfied
  by a published certified adapter. Publication succeeds.

### Invalid

- `i012-hash-conflict.json` — Attempt to publish same name + version
  with different content hash. Expected: `E_HASH_CONFLICT`.
- `i012-missing-interface.json` — Adapter declares `implements:
  "NonExistent@1.0"`. Expected: `E_INTERFACE_NOT_FOUND`.
- `i012-yanked-dependency.json` — Rod depends on a yanked interface.
  Expected: `E_ARTIFACT_YANKED`.

## Alternatives Considered

**Flat registry (no namespaces):** Simpler but name collisions are
inevitable at scale. NPM's scoped packages and Docker's namespace
model both demonstrate the need. Rejected.

**External registry only (e.g., OCI):** Leverages existing
infrastructure but loses OpenStrux-specific validation (structural
compatibility, certification binding). The hub can use OCI as a
storage backend while adding its own validation layer. Deferred to
implementation.

## Consequences

- Hub is a new infrastructure component with storage, validation,
  and governance responsibilities
- All artifact kinds share a common manifest envelope
- Content addressing enables offline builds and tamper detection
- Namespace governance scales from individual to enterprise
- Resolver policy (ADR-009) and policy resolution (ADR-010) are
  concrete operations against this artifact model
- Lock file (ADR-014) pins artifact versions and hashes from this
  registry
