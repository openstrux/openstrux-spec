# OpenStrux v0.4 — Formal Grammar

This section defines the formal grammar for OpenStrux v0.4 in EBNF
notation. Productions are derived from
[syntax-reference.md](syntax-reference.md),
[type-system.md](type-system.md),
[panel-shorthand.md](panel-shorthand.md), and
[expression-shorthand.md](expression-shorthand.md).

Where a production is simplified or incomplete, a `(* TODO: expand *)`
comment marks the gap.

Both verbose and shorthand forms are defined. A conformant parser MUST
accept both forms and normalize them to the same AST (see ADR-006).
The shorthand form is RECOMMENDED for authoring.

---

## 1. Source File

```ebnf
source_file   = { top_level_decl } ;
top_level_decl = type_def | panel_def | context_def ;
```

---

## 2. Type Definitions (`@type`)

Derived from [type-system.md §7](type-system.md).

```ebnf
type_def      = "@type" name "=" "union" "{" union_body "}"
              | "@type" name "=" "enum" "{" enum_body "}"
              | "@type" name "{" record_body "}" ;

union_body    = { variant } ;
variant       = name ":" type_expr ;

enum_body     = name { "," name } ;

record_body   = { field_decl } ;
field_decl    = name ":" type_expr ;

type_expr     = primitive
              | container
              | constrained
              | name ;                      (* user-defined type reference *)

primitive     = "string" | "number" | "bool" | "date" | "bytes" ;

container     = "Optional" "<" type_expr ">"
              | "Batch" "<" type_expr ">"
              | "Map" "<" type_expr "," type_expr ">"
              | "Single" "<" type_expr ">"
              | "Stream" "<" type_expr ">" ;

constrained   = "number" "[" number ".." number "]"
              | "string" "[" string_list "]" ;

string_list   = string { "," string } ;
```

### Type Paths

```ebnf
type_path     = name { "." name } ;
path_match    = type_path | type_path ".*" | "*" ;
```

---

## 3. Panel Definitions (`@panel`)

Derived from [syntax-reference.md](syntax-reference.md) and
[panel-shorthand.md](panel-shorthand.md).

### Verbose Form

```ebnf
panel_def     = "@panel" name "{" panel_body "}" ;
panel_body    = { panel_member } ;
panel_member  = dp_block | access_block | rod_def | snap_stmt ;

dp_block      = "@dp" "{" kv_pairs "}" ;
access_block  = "@access" "{" access_body "}" ;

(* Verbose access: nested intent/scope *)
access_body   = "intent" ":" "{" kv_pairs "}" [ "," "scope" ":" scope_expr ]
              | kv_pairs ;                  (* shorthand: flat fields *)

scope_expr    = "policy" "(" string ")" ;

rod_def       = "@rod" name "=" rod_type "{" rod_body "}"
              | name "=" rod_type "{" rod_body "}" ;  (* shorthand: no @rod *)

rod_type      = "read-data" | "write-data"
              | "receive" | "respond" | "call"
              | "transform" | "filter" | "group" | "aggregate"
              | "merge" | "join" | "window"
              | "guard" | "store"
              | "validate" | "pseudonymize" | "encrypt"
              | "split" ;

rod_body      = { rod_member } ;
rod_member    = knot_assign | snap_stmt | from_clause ;

(* Verbose knot assignment *)
knot_assign   = knot_prefix name "=" value_expr
              | name ":" value_expr ;       (* shorthand: no prefix *)

knot_prefix   = "cfg." | "arg." ;

snap_stmt     = "snap" qualified_knot "->" qualified_knot ;
qualified_knot = name "." knot_dir "." name ;
knot_dir      = "out" | "in" | "err" ;
```

### Shorthand Form (Panel Shorthand Rules 1–4)

```ebnf
(* Rule 1: @rod and = are optional inside @panel *)
(* Rule 2: cfg./arg. prefixes are optional — compiler resolves from rod type *)
(* Rule 3: implicit linear chain — no snap needed for sequential rods *)
(* Rule 4: @access intent fields can be flattened *)

from_clause   = "from" ":" from_ref ;
from_ref      = name "." name             (* rod.knot *)
              | "[" name "," name "]" ;    (* multi-input: [left, right] *)
```

---

## 4. Context Definitions (`@context`)

Derived from [config-inheritance.md](config-inheritance.md).

```ebnf
context_def   = "@context" "{" context_body "}" ;
context_body  = { context_member } ;
context_member = dp_block | access_block | ops_block | sec_block
               | source_def | target_def ;

source_def    = "@source" name "=" type_path "{" kv_pairs "}" ;
target_def    = "@target" name "=" type_path "{" kv_pairs "}" ;

ops_block     = "@ops" "{" kv_pairs "}" ;
sec_block     = "@sec" "{" kv_pairs "}" ;
```

---

## 5. Decorators

```ebnf
cert_block    = "@cert" "{" kv_pairs "}" ;

(* @dp, @access, @ops, @sec defined above *)
(* @cert is never inherited — per-component only *)
```

---

## 6. Value Expressions

```ebnf
value_expr    = string | number | duration | bool | "null"
              | env_ref | secret_ref | source_ref
              | type_path_value | array | object
              | expression ;

duration      = digit+ duration_unit ;
duration_unit = "s" | "m" | "h" | "d" ;
(* Greedy rule: "5m" → duration; "5 m" → number + identifier; single unit only. *)

env_ref       = "env" "(" string ")" ;
secret_ref    = "secret_ref" "{" kv_pairs "}" ;
source_ref    = "@" name [ "{" kv_pairs "}" ] ;   (* named source with optional overrides *)

type_path_value = type_path "{" kv_pairs "}" ;    (* e.g., db.sql.postgres { host: ... } *)

array         = "[" [ value_expr { "," value_expr } ] "]" ;
object        = "{" kv_pairs "}" ;

kv_pairs      = { kv_pair } ;
kv_pair       = name ":" value_expr [ "," ] ;
```

---

## 7. Expression Shorthand

Derived from [expression-shorthand.md](expression-shorthand.md).
Expressions appear as `value_expr` in `arg.*` knot assignments.

```ebnf
expression    = portable_expr | prefixed_expr ;

prefixed_expr = prefix ":" raw_content ;
prefix        = "sql" | "mongo" | "kafka" | "fn" | "opa" | "cedar" ;
raw_content   = (* everything until end of statement *) ;
```

### Filter (arg.predicate)

```ebnf
portable_expr = or_expr ;
or_expr       = and_expr { "OR" and_expr } ;
and_expr      = not_expr { "AND" not_expr } ;
not_expr      = "NOT" atom | atom ;
atom          = compare | in_list | between | is_null | like
              | exists | "(" or_expr ")" ;

compare       = field_ref comp_op value ;
comp_op       = "==" | "!=" | ">" | ">=" | "<" | "<=" ;
in_list       = field_ref [ "NOT" ] "IN" "(" value { "," value } ")" ;
between       = field_ref "BETWEEN" value "AND" value ;
is_null       = field_ref "IS" [ "NOT" ] "NULL" ;
like          = field_ref [ "NOT" ] "LIKE" string ;
exists        = field_ref "EXISTS" ;

field_ref     = name { "." name } ;
value         = string | number | bool | "null" | env_ref ;
string        = '"' chars '"' ;
number        = digit { digit } [ "." digit { digit } ] ;
bool          = "true" | "false" ;
```

### Projection (arg.fields)

```ebnf
projection    = "[" field_entry { "," field_entry } "]" | prefixed_expr ;
field_entry   = "*" | exclude | select | computed ;
exclude       = "-" field_ref ;
select        = field_ref [ "AS" name ] ;
computed      = scalar_expr "AS" name ;
scalar_expr   = field_ref | literal | arithmetic | coalesce | case_when ;
arithmetic    = scalar_expr arith_op scalar_expr ;
arith_op      = "+" | "-" | "*" | "/" | "%" ;
coalesce      = "COALESCE" "(" scalar_expr { "," scalar_expr } ")" ;
case_when     = "CASE" { "WHEN" portable_expr "THEN" scalar_expr }
                "ELSE" scalar_expr "END" ;
```

### Aggregation (arg.fn)

```ebnf
aggregation   = single_agg | multi_agg | prefixed_expr ;
multi_agg     = "[" single_agg { "," single_agg } "]" ;
single_agg    = agg_fn "(" agg_arg ")" [ "AS" name ] ;
agg_fn        = "COUNT" | "SUM" | "AVG" | "MIN" | "MAX"
              | "FIRST" | "LAST" | "COLLECT" ;
agg_arg       = "*" | [ "DISTINCT" ] field_ref ;
```

### Group Key (arg.key)

```ebnf
group_key     = key_entry { "," key_entry } | prefixed_expr ;
key_entry     = field_ref | function_call ;
function_call = name "(" field_ref ")" ;
```

### Join Condition (arg.on)

```ebnf
join_cond     = key_match { "AND" key_match } | prefixed_expr ;
key_match     = qualified_ref "==" qualified_ref ;
qualified_ref = ( "left" | "right" ) "." field_ref ;
```

### Sort (arg.order)

```ebnf
order_expr    = order_field { "," order_field } | prefixed_expr ;
order_field   = field_ref [ "ASC" | "DESC" ] [ "NULLS" ( "FIRST" | "LAST" ) ] ;
```

### Split Routes (arg.routes)

```ebnf
routes        = "{" { route_entry } "}" ;
route_entry   = name ":" ( portable_expr | "*" ) ;
```

### Guard Policy (arg.policy)

```ebnf
policy_expr   = guard_or_expr | prefixed_expr ;
guard_or_expr = guard_and_expr { "OR" guard_and_expr } ;
guard_and_expr = guard_atom { "AND" guard_atom } ;
guard_atom    = context_compare | has_check | atom ;

context_compare = context_ref comp_op value ;
context_ref   = ( "principal" | "intent" | "scope" | "element" ) "." field_ref ;

has_check     = context_ref "HAS" ( value | "ANY" value_list | "ALL" value_list ) ;
value_list    = "(" value { "," value } ")" ;
```

---

## 8. Lexical Rules

```ebnf
name          = letter { letter | digit | "_" | "-" } ;
letter        = "a".."z" | "A".."Z" ;
digit         = "0".."9" ;
number        = digit { digit } [ "." digit { digit } ] ;
comment       = "//" (* everything until end of line *) ;
whitespace    = " " | "\t" | "\n" | "\r" ;
```

**Duration lexical note:** A `duration` token is emitted greedily when a digit sequence is
immediately followed (no whitespace) by one of `s`, `m`, `h`, `d` and that unit character is
not followed by another identifier character. Examples: `5m` → DURATION, `5 m` → NUMBER + IDENT,
`5ms` → NUMBER + IDENT, `30s` → DURATION.

Comments and whitespace are ignored by the parser except within string
literals.

---

## 9. Normalization

A conformant parser MUST normalize all accepted forms to the canonical
AST before IR construction. Specifically:

- `@rod name = type { cfg.field = val }` and `name = type { field: val }`
  produce identical AST nodes (Panel Shorthand Rule 1 + 2).
- Sequential rods without `from:` or `snap` produce implicit snap edges
  using default knots (Panel Shorthand Rule 3).
- Flat `@access` fields are routed to `intent` or `scope` sub-structures
  (Panel Shorthand Rule 4).

See [panel-shorthand.md](panel-shorthand.md) for the complete
normalization rules and ADR-006 for the design rationale.

## Scope

This grammar is normative for OpenStrux v0.4. A conformant parser MUST
accept all productions defined here. Productions marked with
`(* TODO: expand *)` are known incomplete and will be refined in future
spec versions.
