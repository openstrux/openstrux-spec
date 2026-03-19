# ADR-014: Lock File Semantics and Determinism Contract

- **Status:** Accepted
- **Date:** 2026-03-19
- **Context:** The manifesto requires "all shipped generated artifacts
  MUST be reproducible from source plus lock state" (P6 MUST-1) and
  "dependency locks MUST pin versions, hashes, and certification
  snapshots" (P7 MUST-2). Multiple ADRs reference `snap.lock` but no
  ADR owns its schema, resolution rules, or determinism guarantee.
  This ADR is that owner.

## Decision

`snap.lock` is a **deterministic, content-addressed lock file** that
pins every external dependency to an exact version, hash, and
certification snapshot. The invariant is:

> **Same source + same `snap.lock` = same compiled output.**

This is the reproducibility contract. It holds across machines, across
time, and across compiler versions (within the same spec version).

## 1. Lock File Schema

```json
{
  "lock_version": "1",
  "spec_version": "0.5",
  "generated_at": "2026-03-19T10:00:00Z",
  "generated_by": "strux@0.5.0",

  "source_hash": "sha256:aaa...",

  "context_chain": [
    {
      "path": "strux.context",
      "hash": "sha256:bbb..."
    },
    {
      "path": "pipelines/strux.context",
      "hash": "sha256:ccc..."
    }
  ],

  "dependencies": {
    "rods": {
      "read-data": {
        "namespace": "openstrux",
        "version": "1.2.0",
        "hash": "sha256:ddd...",
        "cert": {
          "level": "certified",
          "scope": { "source": "db.sql.*" },
          "hash": "sha256:eee..."
        }
      }
    },

    "adapters": {
      "PostgresConfig -> beam-python": {
        "namespace": "openstrux",
        "version": "2.1.3",
        "hash": "sha256:fff...",
        "interface": "PostgresConfig@1.2",
        "interface_hash": "sha256:ggg...",
        "cert": {
          "level": "certified",
          "scope": { "source": "db.sql.postgres" },
          "expires": "2027-01-15",
          "hash": "sha256:hhh..."
        }
      }
    },

    "policies": {
      "data-engineering-read": {
        "type": "hub",
        "namespace": "openstrux",
        "version": "1.2.0",
        "hash": "sha256:iii...",
        "policy_verification": "static"
      },
      "opa:data.authz.allow": {
        "type": "external",
        "engine": "opa",
        "endpoint": "https://opa.internal/v1/data/authz/allow",
        "policy_verification": "runtime",
        "pinned_at": "2026-03-19T10:00:00Z"
      }
    },

    "interfaces": {
      "PostgresConfig": {
        "namespace": "openstrux",
        "version": "1.2.0",
        "hash": "sha256:jjj..."
      }
    },

    "types": {
      "DataSource": {
        "namespace": "openstrux",
        "version": "1.0.0",
        "hash": "sha256:kkk..."
      }
    }
  }
}
```

### Required Top-Level Fields

| Field | Description |
|---|---|
| `lock_version` | Lock file format version (for forward compat) |
| `spec_version` | OpenStrux spec version |
| `generated_at` | Timestamp of lock generation |
| `generated_by` | Tool + version that generated the lock |
| `source_hash` | Hash of all `.strux` source files in scope |
| `context_chain` | Ordered list of `strux.context` files with hashes |
| `dependencies` | All pinned external artifacts, by kind |

### Per-Dependency Fields

Every dependency entry (regardless of kind) MUST have:

| Field | Description |
|---|---|
| `namespace` | Hub namespace (absent for external policies) |
| `version` | Exact semver (not range) |
| `hash` | Content hash matching hub artifact |

Optional per-dependency:

| Field | Description |
|---|---|
| `cert` | Certification snapshot at lock time |
| `cert.hash` | Hash of the certification record itself |
| `cert.expires` | Certification expiry (if time-bound) |

## 2. What Gets Pinned

| Dependency kind | Pinned fields | Why |
|---|---|---|
| Rod | version, hash, cert snapshot | Reproducibility + audit |
| Adapter | version, hash, interface version, interface hash, cert | Reproducibility + interface compat |
| Interface | version, hash | Adapter compatibility baseline |
| Hub policy | version, hash, policy_verification | Static verification reproducibility |
| External policy | engine, endpoint, policy_verification, pinned_at | Audit trail (content NOT pinned — by design per ADR-010) |
| Type | version, hash | Type definition reproducibility |
| Context chain | path, hash per file | Config inheritance reproducibility |

## 3. Lock Generation

Lock generation (`strux lock`) resolves all dependencies and writes
`snap.lock`:

### Resolution Order

1. **Parse source** — identify all referenced rods, type paths, policy
   names, and named sources
2. **Resolve context chain** — walk directory tree, collect
   `strux.context` files, hash each
3. **Resolve types** — from hub or local definitions, pin version + hash
4. **Resolve rods** — from hub, pin version + hash
5. **Resolve adapters** — for each rod's `adapter-compat` range + target,
   apply resolver policy (ADR-009 §2), select exact version
6. **Resolve interfaces** — for each adapter's `implements`, pin version
7. **Resolve policies** — for hub policies, pin version + hash; for
   external, pin engine ref + endpoint
8. **Snapshot certifications** — for every pinned artifact, record
   current certification status
9. **Hash source** — compute `source_hash` over all `.strux` files
10. **Write lock** — atomic write of `snap.lock`

### Resolver Policy

Resolver policy (from `strux.context` `@resolve` block, ADR-009 §2)
constrains selection before the lock is written. The lock stores the
**result**, not the policy. Once locked, resolver policy is not
re-evaluated.

### Lock Freshness

`strux lock` always generates a fresh lock from current hub state.
`strux lock --frozen` verifies that the existing lock is still valid
(all artifacts exist, no yanked dependencies) without updating it.

## 4. Lock Consumption

### Build (`strux build`)

1. Read `snap.lock`
2. Verify `source_hash` matches current source files. If mismatch →
   `E_LOCK_STALE` (source changed since lock was generated)
3. Verify `context_chain` hashes match current context files. If
   mismatch → `E_LOCK_STALE`
4. For each dependency: fetch artifact by hash (not by version).
   If hash not found in hub or cache → `E_ARTIFACT_NOT_FOUND`
5. Verify fetched artifact hash matches lock hash. If mismatch →
   `E_ARTIFACT_TAMPERED`
6. Proceed with compilation using pinned artifacts

### CI / Audit (`strux build --locked`)

Strict mode. All of the above, plus:

- `E_LOCK_STALE` is fatal (no implicit re-resolution)
- `W_UNCERTIFIED` is promoted to `E_UNCERTIFIED`
- `W_CERT_EXPIRED` is promoted to `E_CERT_EXPIRED`
- No network calls to hub — all artifacts must be in local cache

## 5. Lock Invalidation

The lock becomes stale when:

| Event | Diagnostic | Fix |
|---|---|---|
| Source `.strux` file changed | `E_LOCK_STALE` | `strux lock` |
| `strux.context` file changed | `E_LOCK_STALE` | `strux lock` |
| Dependency yanked in hub | `E_ARTIFACT_YANKED` | `strux lock` (selects replacement) |
| Dependency cert expired | `W_CERT_EXPIRED` | `strux lock` or renew cert |
| Lock file deleted | `E_NO_LOCK` | `strux lock` |
| Lock format version unsupported | `E_LOCK_VERSION` | Upgrade tooling |

The lock is NOT invalidated by:

- New versions available in hub (lock pins exact versions)
- Resolver policy changes (lock stores results, not policy)
- Hub metadata changes that don't affect content hash

## 6. Relationship to Manifest

| Artifact | Purpose | Contains |
|---|---|---|
| `snap.lock` | Reproducibility contract | Dependency versions, hashes, cert snapshots |
| `mf.strux.json` | Compiled output | Fully resolved panel config, IR, lineage |

The lock is an **input** to compilation. The manifest is an **output**.

```
source + snap.lock → compiler → mf.strux.json + emitted code
```

The manifest MAY reference lock entries for traceability (e.g., which
adapter version was used), but the lock is the authoritative source for
dependency state.

## 7. Determinism Guarantee

The determinism contract:

> Given identical `.strux` source files, identical `strux.context`
> files, and identical `snap.lock`, the compiler MUST produce
> byte-identical `mf.strux.json` and semantically identical emitted
> code for the same target.

"Semantically identical" for emitted code means: different whitespace
or comments are acceptable; different logic, different imports, or
different runtime behavior are not.

### What Can Break Determinism

| Source of non-determinism | Mitigation |
|---|---|
| Timestamp in output | Compiler MUST NOT embed generation timestamps in `mf.strux.json` |
| Random ordering | All collections in manifest MUST be sorted deterministically |
| Floating-point | Not applicable (no float computation in compilation) |
| Compiler version | `generated_by` in lock enables version-pinned CI |

## 8. Conformance Cases

### Valid

- `v014-locked-build.strux` — Panel with complete `snap.lock`. Build
  succeeds, output matches golden file.
- `v014-frozen-verify.strux` — `strux lock --frozen` with valid lock.
  Verification passes.

### Invalid

- `i014-no-lock.strux` — `strux build --locked` with no `snap.lock`.
  Expected: `E_NO_LOCK`.
- `i014-stale-source.strux` — Source file changed after lock generation.
  Expected: `E_LOCK_STALE`.
- `i014-stale-context.strux` — Context file changed after lock
  generation. Expected: `E_LOCK_STALE`.
- `i014-tampered-artifact.strux` — Artifact in cache has different hash
  than lock entry. Expected: `E_ARTIFACT_TAMPERED`.
- `i014-yanked-dep.strux` — Lock references a yanked artifact.
  Expected: `E_ARTIFACT_YANKED`.
- `i014-cert-expired.strux` — Locked adapter cert has expired.
  Expected: `W_CERT_EXPIRED` (promoted to `E_CERT_EXPIRED` in
  `--locked` mode).

## Alternatives Considered

**No lock file (always resolve from hub):** Every build fetches latest
versions. Non-reproducible. Rejected — violates P6 MUST-1.

**Lock only versions, not hashes:** Trusts the hub to serve consistent
content for a given version. Insufficient — doesn't detect tampering or
hub bugs. Rejected.

**Lock policy content for external policies:** Would make external
policies reproducible but defeats their purpose (fast policy iteration
without recompilation). Rejected per ADR-010 §3.

## Consequences

- `snap.lock` is a mandatory artifact for production builds
- Development builds (`strux build` without `--locked`) may operate
  without a lock (implicit resolution), but emit `W_NO_LOCK`
- CI pipelines SHOULD use `strux build --locked` for reproducibility
- Lock file SHOULD be committed to version control (like `package-lock.json`)
- Lock generation requires hub access; locked builds do not
- The determinism guarantee enables content-addressed build caching
