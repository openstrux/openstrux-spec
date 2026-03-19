# OpenStrux v0.4 — Configuration Inheritance

## Problem

The reference 5-rod panel is ~336 tokens. Of those, ~145 are
**project-constant**: same @dp controller, same default access policy,
same Postgres connection, same credential provider. Every panel in the
same project repeats them.

```
@dp { controller: "MiEmpresa SL", controller_id: "ES-B12345678", ... }   // ~40 tokens — same for all panels
@access { intent: {...}, scope: policy("data-engineering-read") }          // ~30 tokens — same per team
cfg.source = db.sql.postgres { host, port, db_name, tls, credentials }     // ~60 tokens — same per project
credentials: secret_ref { provider: gcp_secret_manager, path: "..." }      // ~15 tokens — same per env
```

Repeating 145 tokens in every panel violates "token-efficient" and
creates maintenance drift risk.

---

## Solution: `strux.context` Files

A `strux.context` file at any directory level establishes **inherited
defaults** for all panels below it. Panels only declare what differs.

### Resolution Order (nearest wins)

```
project-root/strux.context        ← project-wide defaults
  └── pipelines/strux.context     ← team/domain overrides
       └── panel.strux            ← panel-specific overrides
```

Each level can **set**, **narrow**, or **override** any inheritable
block. This is the same cascade model as `.gitconfig`, `tsconfig.json`,
`.eslintrc`, and Terraform provider blocks.

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

### What's Inheritable

| Block | Inheritable? | Override behavior |
|-------|-------------|-------------------|
| `@dp` | Yes | Panel `@dp` merges with context. Panel fields win |
| `@access` | Yes | Panel `@access` can **narrow** (never widen). If panel omits it, context default applies |
| `@source` | Yes (by alias) | Panel `cfg.source = @production` references named source. Can override fields inline |
| `@target` | Yes (by alias) | Same as @source, for write-data targets |
| `@ops` | Yes | Panel/rod `@ops` merges with context. Nearest wins |
| `@sec` | Yes | Panel `@sec` merges with context |
| `@cert` | No | Certification is per-component, never inherited |

### What's NOT Inheritable

Certification (`@cert`) is never inherited — it must be earned per
component. Rod logic (arg, snap) is never inherited — it's always
panel-specific. This prevents "magic" behavior where panels silently
acquire logic from ancestors.

---

## 2. Panel with Inherited Context

Given the `strux.context` above, the reference panel becomes
(same panel in verbose form: [expression-shorthand.md §11](expression-shorthand.md);
in shorthand: [panel-shorthand.md](panel-shorthand.md)):

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

**15 lines. ~150 tokens.**

vs. the self-contained version: 27 lines, ~336 tokens.
vs. v0.3: 28 lines, ~170 tokens.

The panel carries only what's **unique to this panel**: the record ID,
the purpose, the scan mode, the filter predicate, the fields to
pseudonymize, the target dataset, and the DLQ routing.

---

## 3. Source References (`@name`)

The `@name` syntax references a named source/target from context:

```
cfg.source = @production                          // exact match — use as-is
cfg.target = @analytics { dataset: "eu_users" }   // override fields inline
cfg.target = @events { topic: "user-created" }    // override topic
```

### Resolution Rules

1. `@name` resolves to nearest context that defines it (folder cascade)
2. Inline fields override the resolved source — like spread: `{ ...@analytics, dataset: "eu_users" }`
3. Type is preserved — `@production` is still `db.sql.postgres`, so all type-checking and pushdown work unchanged
4. If `@name` is not found in any ancestor context → compile error

### Named Source vs Inline

| | Named (`@production`) | Inline (`db.sql.postgres { ... }`) |
|---|---|---|
| Token cost | ~3 tokens | ~60 tokens |
| Reusability | All panels share same config | Panel-specific |
| Change propagation | Change context → all panels update | Change each panel |
| Type safety | Same — resolved at compile time | Same |
| Cert scope | Same — type path extracted from resolved source | Same |

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

If a panel specifies `dpo: "other@miempresa.es"`, it overrides the
context value. This is **field-level merge**, not replacement.

---

## 5. `@access` Narrowing

Access context follows the **scope-narrowing** principle from
`access-context.strux`: child scopes can only be **equal or more
restrictive** than parent scopes.

```
// Context @access:
{ intent: { basis: "legitimate_interest" }, scope: policy("default-read") }

// Panel @access:
{ intent: { purpose: "geo_segmentation", operation: "read" } }

// Resolved:
{ intent: { purpose: "geo_segmentation", basis: "legitimate_interest", operation: "read" }, scope: policy("default-read") }
```

Compile errors if panel tries to widen:

```
// Context scope: policy("default-read")
// Panel scope: policy("admin-full-access")  ← COMPILE ERROR: widening scope
```

The compiler knows policy definitions and can verify that one policy
is a subset of another. If it can't determine the relationship, it
emits a warning.

---

## 6. Folder-Level Overrides

Teams or domains can have their own `strux.context`:

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
This mirrors how organizations actually structure configuration:
company → team → service.

---

## 7. Compiled Output

The compiled manifest (`mf.strux.json`) always contains the **fully
resolved** configuration. No inheritance at runtime — the compiler
flattens everything:

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

This means:

- **Audit** always sees the full picture — no hidden inheritance
- **Certification** evaluates the resolved config, not the source shorthand
- **Runtime** has no concept of inheritance — pure flat config
- **Source** is compact; **compiled** is complete. Same principle as expression shorthand.

---

## 8. `strux context resolve` CLI

```bash
# Show resolved context for a panel
strux context resolve pipelines/user-analytics.strux

# Show what a panel inherits (diff from self-contained)
strux context resolve --show-inherited pipelines/user-analytics.strux

# Validate context cascade (no cycles, no widening, all @name resolved)
strux context check
```

---

## 9. State-of-Art Comparison

| System | Mechanism | OpenStrux equivalent |
|--------|-----------|---------------------|
| **Terraform** | `provider` blocks at module level, inherited by resources | `@source`/`@target` in `strux.context` |
| **dbt** | `dbt_project.yml` + `profiles.yml` (project defaults + environment) | `strux.context` cascade |
| **ESLint/TSConfig** | Folder cascade with `extends` | Folder-level `strux.context` |
| **Kubernetes** | Namespace defaults + pod overrides, LimitRange, ResourceQuota | `@ops` inheritance, `@access` narrowing |
| **CSS** | Cascade specificity (global → element → inline) | project → folder → panel |
| **Git** | `.gitconfig` system → global → project | Same cascade model |
| **Helm** | `values.yaml` with override hierarchy | Named sources with inline overrides |

### What OpenStrux adds

**Scope narrowing is enforced.** In Terraform, a module can reference
any provider. In Kubernetes, a pod can request more resources than the
namespace allows (and gets rejected at admission). In OpenStrux, the
compiler proves at build time that child `@access` scopes are subsets
of parent scopes. This is a type-system guarantee, not a runtime check.

---

## 10. Impact on Token Budget

| Metric | Self-contained | With context | Savings |
|--------|---------------|-------------|---------|
| Panel lines | 27 | 15 | -44% |
| Panel tokens | ~336 | ~150 | -55% |
| Context tokens (amortized per panel, 10 panels) | 0 | ~25 | +25 |
| **Effective tokens per panel** | **~336** | **~175** | **-48%** |

At 10 panels per project, the context file costs ~250 tokens total,
or ~25 amortized per panel. The net saving is ~161 tokens per panel.

### Updated Benchmark Comparison

| Variant | Lines | Tokens | Ratio vs Beam Python |
|---------|-------|--------|---------------------|
| v0.3 panel | 28 | ~170 | ~0.46 |
| v0.4 self-contained (normal) | 27 | ~336 | ~0.92 |
| **v0.4 with context** | **15** | **~150** | **~0.41** |
| v0.4 with context (amortized) | 15 | ~175 | ~0.48 |
| Generated Beam Python | 40 | ~366 | 1.00 |

With multi-target generation (Beam + TS + compliance), the amortized
ratio would be ~0.12. Well under the 0.25 MUST threshold.

---

## 11. Design Principles

### Source is compact. Compiled is complete

This is the same principle everywhere in v0.4:

- **Expressions**: shorthand (compact) → AST (complete)
- **Config**: context-inherited (compact) → resolved manifest (complete)
- **Types**: union path `db.sql.postgres` (compact) → full PostgresConfig (complete)

The source form is what humans and LLMs write. The compiled form is
what machines, auditors, and runtimes consume.

### Fail-closed inheritance

- If no `strux.context` exists → panel must be self-contained (v0.4 as-is)
- If `@name` reference not found → compile error (not silent empty)
- If `@access` widening detected → compile error
- If `@dp` missing required fields after merge → compile error

No silent defaults. No magic. Every resolved value traces back to an
explicit declaration in some `strux.context` or panel file.

### Context is not code

`strux.context` declares **what** (data sources, policies, metadata).
It never declares **how** (no rods, no snaps, no logic). The graph
topology is always in the panel. This prevents "action at a distance"
where inherited context changes behavior.
