# Task: Fill stub spec files and add conformance fixtures in openstrux-spec

## Context

You are working in the `openstrux-spec` repository. Seven spec files in
`specs/core/` are empty stubs that block the pre-push hook. You must fill
them with accurate, normative spec content derived from the existing rich
spec material. You must also add at least one valid and one invalid `.strux`
conformance fixture.

**Working rule:** create a worktree on a new branch before touching any file.

```sh
cd /home/homofaber/proj/openstrux-spec
git worktree add ../openstrux-spec-fill-stubs feat/fill-core-stubs
```

Do all work inside `../openstrux-spec-fill-stubs`. Do not push — the human
reviews before any push.

---

## Step 1 — Read ALL existing spec material before writing anything

Read every file listed below in full. Understand what is already defined
before writing a single word in a stub. Do not duplicate content that already
exists in another spec file — cross-reference it instead.

**Read these files:**

- `specs/core/syntax-reference.md` — compact grammar and construct overview
- `specs/core/type-system.md` — union, record, enum, type narrowing, type paths
- `specs/core/expression-shorthand.md` — expression forms, access expressions
- `specs/core/config-inheritance.md` — context inheritance, strux.context
- `specs/core/design-notes.md` — design rationale and architectural decisions
- `specs/core/panel-shorthand.md` — panel and rod shorthand forms
- `specs/core/access-context.strux` — AccessContext type definition
- `specs/core/expressions.strux` — expression type definitions
- `specs/core/datasource-hierarchy.strux` — DataSource union hierarchy
- `specs/modules/rods/overview.md` — 18 basic rod taxonomy
- `specs/profiles/mvp.md` — MVP profile constraints
- `adr/` — all ADRs (architectural decisions)

---

## Step 2 — Fill the 7 stub files

Each file must:
- Remove the `> Status: stub` line and the placeholder HTML comments
- Be accurate and consistent with the existing spec material
- Cross-reference other spec files rather than re-explaining what they define
- Be normative in tone ("MUST", "SHALL", "MAY" per RFC 2119 where appropriate)
- Not invent new language features — only document what the existing specs define

### `specs/core/overview.md`

Write a concise language overview (300–600 words). Cover:
- What Openstrux is and what problem it solves
- The three construct layers: strux types, panels, rods
- The core design principles (from design-notes.md: structure first, determinism, explicit types)
- What a valid Openstrux source file looks like at a high level
- What it does NOT do (guard against misunderstanding)
- Reference the other core spec files for detail; do not duplicate them

### `specs/core/terminology.md`

Write a glossary. Every term must be defined precisely. At minimum define:
`strux`, `panel`, `rod`, `knot` (cfg/arg/in/out/err), `type path`, `union`,
`discriminant`, `AccessContext`, `snap`, `lock`, `manifest`, `conformance fixture`,
`target`, `emission`, `pipeline`, `DataSource`, `strux.context`, `change package`.

Format: term in bold, one-paragraph definition, cross-reference to the
authoritative spec section.

### `specs/core/grammar.md`

Write a formal grammar for Openstrux v0.4. Derive it from `syntax-reference.md`
and the `.strux` example files. Use EBNF. Cover at minimum:

- Top-level declarations (`@strux`, `@panel`, `@access`, `@dp`)
- Record, enum, union forms
- Field declarations and type expressions (primitives, containers, constraints)
- Type paths (dot notation, wildcards)
- Rod declarations inside panels
- Knot expressions (cfg/arg/in/out/err)
- Access expressions (`@access { intent, scope }`)
- Expression shorthand forms (from expression-shorthand.md)

Mark any productions that are simplified or incomplete with a `(* TODO: expand *)`
comment so gaps are visible rather than hidden.

### `specs/core/ir.md`

Write the Intermediate Representation (IR) specification. Cover:
- What the IR is and when it is produced (after parse, before emission)
- IR node types corresponding to strux constructs (derive from type-system.md AST)
- How union narrowing is represented in the IR
- How panels and rods are represented (derive from rods/overview.md)
- How AccessContext is attached to IR nodes
- What invariants the IR must satisfy (e.g. all type paths resolved, no ambiguous unions)
- What the IR does NOT contain (e.g. emission-target specifics)

Cross-reference `openstrux-core/packages/ast/` for the implementation-level
TypeScript interfaces (read those files if they exist).

### `specs/core/semantics.md`

Write the evaluation semantics. Cover:
- Execution model: how a panel executes (rod chain evaluation order)
- Knot data flow: how data flows from rod output knots to the next rod's input knots
- Implicit vs explicit chaining (from panel-shorthand.md)
- Type narrowing semantics at evaluation time
- AccessContext evaluation: when and how access constraints are checked
- Error propagation: how rod `err` knots propagate
- Determinism requirement: same source + same lock → same output (reference locks.md)
- What is NOT specified here (target-specific behaviour, network semantics)

### `specs/core/locks.md`

Write the lock semantics. Cover:
- What a lock is and why it exists (determinism guarantee)
- What `snap.lock` contains (dependency versions, content hashes, resolution state)
- Lock creation: when it is generated, what inputs determine it
- Lock consumption: how a build system uses the lock to reproduce an output
- Lock invalidation: what changes force a lock update
- Relationship between lock and manifest (`mf.strux.json`)
- Conformance requirement: a compliant implementation MUST produce identical
  output given identical source and identical lock

### `specs/core/conformance.md`

Write the conformance specification. Cover:
- Conformance classes: what it means for a parser, validator, and emitter to
  be conformant with this spec
- Normative vs informative sections (define which sections of this spec are normative)
- Conformance fixtures: valid fixtures MUST parse and typecheck clean; invalid
  fixtures MUST fail with the specified diagnostic code
- Golden fixtures: a conformant emitter MUST produce output matching the golden file
- Levels: what a minimal conformant implementation MUST support vs MAY omit
- Reference to `conformance/` directory for the fixture set
- How to report non-conformance (diagnostic format)

---

## Step 3 — Add conformance fixtures

Create at minimum:

**`conformance/valid/v001-record-basic.strux`**
A minimal valid `.strux` file defining one record type with two or three
primitive fields. Must parse and typecheck clean. Add a comment header:
```
// fixture: v001-record-basic
// expect: valid
// covers: record declaration, primitive field types
```

**`conformance/valid/v002-union-narrowing.strux`**
A valid `.strux` file defining a simple union type and a panel that uses it
with type narrowing via a dot-path. Must parse and typecheck clean.

**`conformance/invalid/i001-unknown-type.strux`**
An invalid `.strux` file that references a type name that is not declared.
Must fail. Add:
```
// fixture: i001-unknown-type
// expect: invalid
// error-code: E_UNKNOWN_TYPE
// covers: type resolution failure
```

**`conformance/invalid/i002-missing-required-field.strux`**
An invalid `.strux` file that omits a required field in a record.
Must fail with a type error.

Derive the syntax for all fixtures from `syntax-reference.md` and the
existing `.strux` example files. Do not invent syntax — only use forms that
are already specified.

---

## Step 4 — Review and self-check before stopping

Before finishing, do the following:

1. Run the pre-push hook manually to see if the checks pass:
   ```sh
   bash .githooks/pre-push
   ```
   Fix any remaining failures.

2. Verify: grep for `Status: stub` in `specs/` — result must be zero.

3. Verify: count `.strux` files in `conformance/valid/` and `conformance/invalid/`
   — must be at least 1 each.

4. Read each filled stub file once more and check:
   - No content contradicts `syntax-reference.md` or `type-system.md`
   - No new language features invented
   - Cross-references point to real files that exist
   - Tone is normative where appropriate

5. Do NOT push. Report what you wrote, any gaps you found, and any decisions
   you made that the human should review.

---

## Constraints

- Work only inside the worktree `../openstrux-spec-fill-stubs`
- Do not modify any file outside `specs/core/` and `conformance/`
- Do not invent Openstrux syntax — derive everything from existing spec files
- Do not push — human reviews first
- If a stub cannot be filled accurately from existing material, write what can
  be derived and add a clearly marked `<!-- GAP: ... -->` comment for human review
