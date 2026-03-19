# OpenStrux v0.4 — Panel Shorthand

## Problem

The context-inherited panel (18 lines, ~209 tokens) still carries
structural overhead that doesn't convey meaning:

```
  @rod f = filter {                          // "@rod" and "=" are ceremony
    arg.predicate = address.country IN ...   // "arg." is redundant (compiler knows)
    snap db.out.rows -> in.data              // snap is predictable (linear chain)
  }
```

~67 tokens (32%) are structural: `@rod`, `=`, `cfg.`, `arg.`, `snap`,
`out.`, `in.`, `->`. All resolvable by the compiler from the rod type
definition and declaration order.

---

## Four Rules

### Rule 1: Drop `@rod` and `=`

Inside a `@panel` block, every statement that isn't `@dp`/`@access`
is a rod. The `@rod` keyword is redundant. Rod names use `=` to
separate name from type, but `:` is already the assignment operator
in strux records. Unify:

```
// Before:
@rod db = read-data { cfg.source = @production, cfg.mode = "scan" }

// After:
db = read-data { source: @production, mode: "scan" }
```

The `@rod` prefix remains valid (backward compatible) but is not
required in panels.

### Rule 2: Drop `cfg.` / `arg.` prefixes

The rod type definition declares which knots are cfg and which are
arg. Inside a rod block, the compiler resolves the prefix from context:

```
// Before:
cfg.algo = "sha256", arg.fields = ["full_name", "email"]

// After:
algo: "sha256", fields: ["full_name", "email"]
```

Resolution: the compiler looks up the rod type definition for
`pseudonymize` and finds `cfg: algo` and `arg: fields`. No ambiguity.

If a custom rod has overlapping names (cfg.x and arg.x), the compiler
emits an error and requires explicit prefix.

### Rule 3: Implicit linear snaps

Rods in declaration order form an **implicit chain**: each rod reads
from the **default output** of the preceding rod into its own
**default input**.

```
db = read-data { source: @production, mode: "scan" }
f = filter { predicate: address.country IN ("ES", "FR", "DE") AND deleted_at IS NULL }
//   ↑ implicit: db.rows -> f.data
```

**Default knots** (defined per rod type):

| Rod | Default out | Default in |
|-----|------------|------------|
| read-data | rows (db) / elements (stream) | — (source) |
| filter | match | data |
| transform | data | data |
| group | grouped | data |
| aggregate | result | grouped |
| pseudonymize | masked | data |
| validate | valid | data |
| encrypt | encrypted | data |
| merge | merged | left (+ right) |
| join | joined | left (+ right) |
| split | (named routes) | data |
| write-data | receipt | rows / elements |
| receive | request | — (trigger) |
| respond | sent | data |
| call | response | request |
| guard | allowed | data |
| store | result | key |
| window | windowed | data |

When the implicit chain doesn't work (branching, merging, or
non-default knots), use `from:`:

```
dlq = write-data { target: @dlq, from: f.reject }
//   ↑ explicit: f.reject -> dlq (not the default chain)
```

`from: rod.knot` is shorthand for `snap rod.out.knot -> in.{default}`.
The `out.` prefix is dropped (you always read from outputs).

For multi-input rods (join, merge):
```
j = join { mode: "inner", on: left.id == right.id, from: [users, orders] }
//   ↑ first = left, second = right
```

### Rule 4: Flatten @access

Drop the `intent:` wrapper when setting intent fields directly:

```
// Before:
@access { intent: { purpose: "geo_segmentation", operation: "read" } }

// After:
@access { purpose: "geo_segmentation", operation: "read" }
```

The compiler recognizes intent fields (purpose, basis, operation,
urgency) and scope fields (scope, resources) and routes them to the
correct sub-structure. If there's ambiguity, use the full form.

---

## Result

```
@panel user-analytics {
  @dp { record: "RPA-2026-003" }
  @access { purpose: "geo_segmentation", operation: "read" }
  db = read-data { source: @production, mode: "scan" }
  f = filter { predicate: address.country IN ("ES", "FR", "DE") AND deleted_at IS NULL }
  p = pseudonymize { algo: "sha256", fields: ["full_name", "email", "national_id"] }
  sink = write-data { target: @analytics { dataset: "eu_users" } }
  dlq = write-data { target: @dlq, from: f.reject }
}
```

**9 lines. ~142 tokens.**

---

## Comparison

| Variant | Lines | Tokens | Context needed |
|---------|-------|--------|----------------|
| v0.3 panel | 28 | ~170 | No |
| v0.4 self-contained (verbose) | 39 | ~325 | No |
| v0.4 self-contained (normal) | 27 | ~336 | No |
| v0.4 with context (current syntax) | 18 | ~209 | Yes |
| **v0.4 with context + shorthand** | **9** | **~142** | Yes |
| v0.4 shorthand, amortized (10 panels) | 9 | ~160 | Yes |

At 142 tokens the panel is **17% more compact than v0.3** while
carrying access control, typed credentials, expression pushdown,
and GDPR compliance — none of which v0.3 had.

---

## Backward Compatibility

All four rules are **syntactic sugar**. The verbose form remains valid:

```
// These are equivalent:
@rod f = filter { arg.predicate = ..., snap db.out.rows -> in.data }
f = filter { predicate: ..., from: db.rows }
f = filter { predicate: ... }    // implicit chain from previous rod
```

The compiler normalizes all forms to the same AST. The compiled
manifest always uses the fully qualified form (with `cfg.`, `arg.`,
explicit snaps) for audit and certification.

---

## Implicit Chain Rules (Formal)

1. **First rod** in a panel has no implicit input (it's a source:
   read-data, receive, etc.).

2. **Subsequent rods** without `from:` read from the **default output**
   of the immediately preceding rod.

3. **`from: rod.knot`** overrides the implicit chain. The rod reads
   from the specified output knot.

4. **`from: [rod1, rod2]`** for multi-input rods. First = left/primary,
   second = right/secondary.

5. **Multiple outputs** (filter.match + filter.reject, split.{routes}):
   the implicit chain always follows the **default** (match, not reject).
   Non-default outputs require explicit `from:` on downstream rods.

6. **No implicit chain across branches.** After a rod with multiple
   downstreams (split, filter with DLQ), the next rod in declaration
   order follows the default output. Other outputs must be wired
   explicitly.

7. **Implicit chain is declaration order, not topological order.** The
   panel author controls the chain by ordering rod declarations. The
   compiler validates that the resulting graph is acyclic.
