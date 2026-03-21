## ADDED Requirements

### Requirement: All 18 rod types are handled by the TypeScript adapter
The TypeScript adapter SHALL register an emitter for every rod type in the 18-rod taxonomy. Encountering an unregistered rod type SHALL produce a `// STRUX-STUB` comment, not a crash or invalid TypeScript.

#### Scenario: Unknown rod does not crash the generator
- **WHEN** the generator processes a panel containing any of the 18 known rod types
- **THEN** `generate()` SHALL return a `GeneratedFile[]` with no thrown exceptions and all files SHALL be valid TypeScript

#### Scenario: Tier 2 stub is grep-able
- **WHEN** a Tier 2 rod (group, aggregate, merge, join, window) appears in a panel
- **THEN** the generated output SHALL contain the literal string `STRUX-STUB` at the corresponding call site

### Requirement: transform rod emits a typed mapping function stub
The `transform` rod emitter SHALL produce a TypeScript function with input and output types inferred from the rod's `in`/`out` knots. The function body SHALL contain a TODO comment preserving the expression text.

#### Scenario: transform with resolved types produces typed stub
- **WHEN** a `transform` rod has `in` knot type `Proposal` and `out` knot type `EligibilityRecord`
- **THEN** the generated code SHALL contain `function transform(input: Proposal): EligibilityRecord {`

### Requirement: write-data rod emits a Prisma mutation stub
The `write-data` rod emitter SHALL produce a Prisma `create` or `update` call stub, with the model name inferred from `cfg.target`.

#### Scenario: write-data produces a Prisma create call
- **WHEN** a `write-data` rod has `cfg.target = db.sql.postgres` and a record type
- **THEN** the generated code SHALL contain `await prisma.<model>.create({ data: input })`

### Requirement: split rod emits a switch-statement routing stub
The `split` rod emitter SHALL produce a TypeScript `switch` block with one `case` per named route in the rod's split config.

#### Scenario: split with two routes produces two cases
- **WHEN** a `split` rod defines routes `{ eligible: ..., ineligible: ... }`
- **THEN** the generated code SHALL contain `case "eligible":` and `case "ineligible":` blocks

### Requirement: pseudonymize and encrypt rods emit compliance stubs with JSDoc
The `pseudonymize` and `encrypt` rod emitters SHALL produce wrapper function stubs with JSDoc comments citing the relevant AccessContext scope field.

#### Scenario: pseudonymize stub references the access scope
- **WHEN** the panel's `@access` block specifies `scope: { fields: ["email", "name"] }`
- **THEN** the generated pseudonymize stub SHALL include a JSDoc comment referencing `email` and `name`
