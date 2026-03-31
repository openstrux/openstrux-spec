# OpenStrux v0.4 — Configuration Inheritance

`strux.context` files establish inherited defaults at any directory level.
Panels only declare what differs from context.

---

## 1. `strux.context` Syntax

```
@context {

  // ── Data Protection (inherited by all panels) ──────────────
  @dp {
    controller:    "MiEmpresa SL"
    controller_id: "ES-B12345678"
    dpo:           "dpo@miempresa.es"
  }

  // ── Default Access (inherited, panels can narrow) ──────────
  @access {
    intent: { basis: "legitimate_interest" }
    scope:  policy("default-read")
  }

  // ── Named Sources (reusable by alias) ──────────────────────
  @source production = db.sql.postgres {
    host:        env("DB_HOST")
    port:        5432
    db_name:     "production"
    tls:         true
    credentials: secret_ref { provider: gcp_secret_manager, path: "projects/mi/secrets/pg" }
  }

  @source analytics = db.sql.bigquery {
    project:  "miempresa-analytics"
    location: "EU"
    credentials: adc {}
  }

  @source events = stream.pubsub {
    project: "miempresa"
  }

  @source dlq = stream.pubsub {
    project: "miempresa"
    topic:   "dlq"
  }

  // ── Default Ops (inherited by all rods) ────────────────────
  @ops {
    retry:    3
    timeout:  "30s"
  }
}
```

### Inheritance Table

| Block | Inheritable? | Override behavior |
|-------|-------------|-------------------|
| `@dp` | Yes | Field-level merge, panel wins |
| `@access` | Yes | Panel can **narrow** (never widen). Omitted → context default applies |
| `@source` | Yes (by alias) | `cfg.source = @production` references named source. Can override fields inline |
| `@target` | Yes (by alias) | Same as @source, for write-data targets |
| `@ops` | Yes | Field-level merge, nearest wins |
| `@sec` | Yes | Field-level merge |
| `@privacy` | Yes | Framework can **narrow** (e.g., `gdpr` → `gdpr.bdsg`). Panel wins on `dpa_ref` |
| `@cert` | No | Per-component only, never inherited |

`@cert` is never inherited — it must be earned per component. Rod logic
(arg, snap) is never inherited — it's always panel-specific.

---

## 2. Panel with Inherited Context

Given the `strux.context` above:

```
@panel user-analytics {
  @dp { record: "RPA-2026-003" }
  @access { intent: { purpose: "geo_segmentation", operation: "read" } }
  @rod db = read-data { cfg.source = @production, cfg.mode = "scan" }
  @rod f = filter {
    arg.predicate = address.country IN ("ES", "FR", "DE") AND deleted_at IS NULL
    snap db.out.rows -> in.data
  }
  @rod p = pseudonymize {
    cfg.algo = "sha256", arg.fields = ["full_name", "email", "national_id"]
    snap f.out.match -> in.data
  }
  @rod sink = write-data {
    cfg.target = @analytics { dataset: "eu_users" }
    snap p.out.masked -> in.rows
  }
  @rod dlq = write-data { cfg.target = @dlq, snap f.out.reject -> in.elements }
}
```

The panel carries only what's unique: record ID, purpose, scan mode,
filter predicate, pseudonymization fields, target dataset, and DLQ routing.

---

## 3. Source References (`@name`)

```
cfg.source = @production                          // exact match — use as-is
cfg.target = @analytics { dataset: "eu_users" }   // override fields inline
cfg.target = @events { topic: "user-created" }    // override topic
```

### Resolution Rules

1. `@name` resolves to nearest context that defines it (folder cascade)
2. Inline fields override the resolved source — like spread: `{ ...@analytics, dataset: "eu_users" }`
3. Type is preserved — `@production` is still `db.sql.postgres`, so type-checking and pushdown work unchanged
4. If `@name` is not found in any ancestor context → compile error

---

## 4. `@dp` Merge Semantics

Panel `@dp` **merges** with context `@dp`. Panel fields win on conflict:

```
// Context @dp:
{ controller: "MiEmpresa SL", controller_id: "ES-B12345678", dpo: "dpo@miempresa.es" }

// Panel @dp:
{ record: "RPA-2026-003" }

// Resolved (at compile time):
{ controller: "MiEmpresa SL", controller_id: "ES-B12345678", dpo: "dpo@miempresa.es", record: "RPA-2026-003" }
```

Field-level merge, not replacement.

---

## 5. `@access` Narrowing

Child scopes can only be **equal or more restrictive** than parent scopes.

```
// Context @access:
{ intent: { basis: "legitimate_interest" }, scope: policy("default-read") }

// Panel @access:
{ intent: { purpose: "geo_segmentation", operation: "read" } }

// Resolved:
{ intent: { purpose: "geo_segmentation", basis: "legitimate_interest", operation: "read" }, scope: policy("default-read") }
```

Widening is a compile error:

```
// Context scope: policy("default-read")
// Panel scope: policy("admin-full-access")  ← COMPILE ERROR: widening scope
```

The compiler verifies policy subset relationships. If it can't determine
the relationship, it emits a warning.

---

## 6. Folder-Level Overrides

```
project-root/
├── strux.context                    # company-wide: @dp, @source production
├── pipelines/
│   ├── strux.context                # data team: @access for data-engineering
│   ├── user-analytics.strux
│   └── revenue-report.strux
├── services/
│   ├── strux.context                # backend team: @ops defaults, @source api-gateway
│   ├── user-api.strux
│   └── order-api.strux
└── compliance/
    ├── strux.context                # compliance team: stricter @access, @dp additions
    └── gdpr-export.strux
```

Each `strux.context` inherits from its parent, then sets/overrides.

---

## 7. Compiled Output

The compiled manifest (`mf.strux.json`) contains the **fully resolved**
configuration. No inheritance at runtime — the compiler flattens everything:

```json
{
  "panel": "user-analytics",
  "dp": {
    "controller": "MiEmpresa SL",
    "controller_id": "ES-B12345678",
    "dpo": "dpo@miempresa.es",
    "record": "RPA-2026-003"
  },
  "access": {
    "intent": { "purpose": "geo_segmentation", "basis": "legitimate_interest", "operation": "read" },
    "scope": "default-read"
  },
  "rods": {
    "db": {
      "type": "read-data",
      "cfg": {
        "source": {
          "type_path": "db.sql.postgres",
          "host": "...",
          "port": 5432,
          "db_name": "production",
          "tls": true
        }
      }
    }
  }
}
```

- **Audit** always sees the full picture — no hidden inheritance
- **Certification** evaluates the resolved config, not the source shorthand
- **Runtime** has no concept of inheritance — pure flat config

---

## 8. `@privacy` Decorator

`@privacy` is a panel-level decorator that declares the governing privacy framework.
It is the regulatory companion to `@dp` (which declares the controller identity):
`@dp` says *who* the controller is; `@privacy` says *which law* governs the processing.

### Syntax

```
@privacy {
  framework: gdpr                    // required
  dpa_ref:   "DPA-2026-001"          // optional: supervisory authority reference
}
```

Or at context level (inherited by all panels in scope):

```
@context {
  @privacy {
    framework: gdpr
  }
  ...
}
```

### Inheritance rules

| Block | Inheritable? | Override behavior |
|---|---|---|
| `@privacy` | Yes | Framework can **narrow** (e.g., `gdpr` → `gdpr.bdsg`). Widening is a compile error. Panel wins on `dpa_ref`. |

A panel that inherits `@privacy { framework: gdpr }` from context may override with
`@privacy { framework: gdpr.bdsg }` (narrower). It MUST NOT override with a sibling framework
(e.g., `ccpa`) — that is a compile error.

### Validation rule: @privacy path coverage

When `@privacy` is declared on a panel (directly or via inheritance), the validator SHALL enforce
the following for every data flow path from a **source rod** (`receive`, `read-data`) to a
**sink rod** (`write-data`, `respond`):

> The path MUST pass through at least one `private-data` rod whose `framework` is compatible
> with the declared `@privacy` framework.

**Compatibility:** `gdpr.bdsg` is compatible with `@privacy { framework: gdpr }` (narrowing is
allowed). `gdpr` is NOT compatible with `@privacy { framework: gdpr.bdsg }` (widening is a
compile error on the path).

**Diagnostic:** If a path bypasses `private-data`, the compiler emits:

```
E_PRIVACY_PATH_BYPASS: data flow path from <source> to <sink> in panel <name>
  does not pass through a private-data rod.
  Panel declares @privacy { framework: gdpr }.
  Add a private-data rod on this path or remove @privacy if this path is intentional.
```

### `@privacy` fields

| Field | Type | Required | Description |
|---|---|---|---|
| `framework` | `PrivacyFramework` type path | Yes | Which law governs the panel |
| `dpa_ref` | `string` | No | Supervisory authority or DPA registration reference |

### Relationship to `@dp`

`@dp` and `@privacy` are complementary, not alternatives:

- `@dp` declares the controller identity, DPO, and record reference.
- `@privacy` declares the regulatory framework in force.
- Both are typically inherited from `strux.context` at project or domain level.
- A panel with `@privacy` but no `@dp` → compile warning (not error): controller identity is
  needed for Art. 30 records.

---

## 9. `strux context resolve` CLI

```bash
# Show resolved context for a panel
strux context resolve pipelines/user-analytics.strux

# Show what a panel inherits (diff from self-contained)
strux context resolve --show-inherited pipelines/user-analytics.strux

# Validate context cascade (no cycles, no widening, all @name resolved)
strux context check
```
