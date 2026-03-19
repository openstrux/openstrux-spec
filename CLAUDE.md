# openstrux-spec — Language Specification

Normative specification for Openstrux v0.4. All spec changes must go through a change package in `changes/`.

## Folder guide

```
specs/core/             Core language: type-system, grammar, semantics, IR, locks, conformance,
                        access-context, expressions, expression-shorthand, syntax-reference,
                        config-inheritance, panel-shorthand, design-notes
specs/modules/          Module specs: datasource-hierarchy, rods/ (overview + per-rod), hub, audit,
                        target-beam, target-ts
specs/profiles/         Deployment profiles: mvp, future
conformance/valid/      .strux fixtures that MUST parse and typecheck cleanly
conformance/invalid/    .strux fixtures that MUST fail with specific diagnostics
conformance/golden/     Fixtures with expected generated output for diff comparison
adr/                    Architecture Decision Records (numbered, dated, status field)
reviews/                Manifesto PASS/WARN/FAIL review reports per spec version (e.g. v0.4-draft.md)
changes/                AI-built change packages — see changes/README.md for format
```

## Change workflow

1. New work → `changes/YYYY-NNN-title/` with `proposal.md`, `design.md`, `tasks.md`, `spec-delta/`, `acceptance.md`
2. On acceptance → merge spec-delta files into `specs/` modules
3. Archive the change package (keep for traceability)

## File conventions

- Strux source definitions → `.strux` extension
- Prose specification documents → `.md`
- Conformance fixtures → `.strux` in `conformance/valid|invalid|golden/`
- Never write generated artifacts into this repo

## Key files

- `specs/core/syntax-reference.md` — compact LLM system prompt reference (~2,000 tokens); update whenever syntax changes
- `specs/core/type-system.md` — union/record/enum forms, type paths, adapters
- `specs/modules/rods/overview.md` — 18 basic rod taxonomy
- `reviews/v0.4-draft.md` — current manifesto review scorecard
