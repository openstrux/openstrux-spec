# OpenStrux v0.4 — Panel Shorthand

Four syntactic-sugar rules that reduce structural ceremony in panels.
Both verbose and shorthand forms normalize to the same AST.

---

## Rule 1: `@rod` keyword optional

Inside a `@panel` block, every statement that isn't `@dp`/`@access`
is a rod. The `@rod` keyword is redundant:

```
// Verbose:
@rod db = read-data { cfg.source = @production, cfg.mode = "scan" }

// Shorthand:
db = read-data { source: @production, mode: "scan" }
```

`@rod` remains valid (backward compatible) but is not required.

## Rule 2: `cfg.` / `arg.` prefixes optional

The compiler resolves prefixes from the rod type definition:

```
// Verbose:
cfg.algo = "sha256", arg.fields = ["full_name", "email"]

// Shorthand:
algo: "sha256", fields: ["full_name", "email"]
```

If a custom rod has overlapping names (cfg.x and arg.x), the compiler
emits an error and requires explicit prefix.

## Rule 3: Implicit linear snaps

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

For multi-input rods (join, merge):

```
j = join { mode: "inner", on: left.id == right.id, from: [users, orders] }
//   ↑ first = left, second = right
```

## Rule 4: Flatten @access

Drop the `intent:` wrapper when setting intent fields directly:

```
// Verbose:
@access { intent: { purpose: "geo_segmentation", operation: "read" } }

// Shorthand:
@access { purpose: "geo_segmentation", operation: "read" }
```

The compiler recognizes intent fields (purpose, basis, operation,
urgency) and scope fields (scope, resources) and routes them to the
correct sub-structure.

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
manifest always uses the fully qualified form for audit and certification.

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

8. **`from:` namespace resolution.** `rod` resolves against rod names
   declared in the same panel. `knot` resolves against the named output
   knot of that rod type (e.g., `out.match`, `out.rows`). Cross-panel
   references are not allowed — compiler MUST emit an error. The
   `from: [rod1, rod2]` form applies the same resolution independently.
