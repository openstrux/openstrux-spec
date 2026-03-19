# openstrux-spec

Normative language and platform specification for Openstrux v0.4.

## Layout

specs/
core/ ← type-system, grammar, semantics, IR, locks, conformance,
expressions, syntax-reference, config-inheritance, panel-shorthand
modules/ ← rods, hub, audit, target-beam, target-ts
profiles/ ← mvp, future
conformance/ ← valid/, invalid/, golden/ fixtures
adr/ ← Architecture Decision Records
reviews/ ← Manifesto reviews per version
changes/ ← AI-built change packages

text

## Current version

`v0.4-draft`

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) and the change package process in [changes/README.md](changes/README.md).

## Local setup

Activate the pre-push validation hook after cloning:

```bash
git config core.hooksPath .githooks
```

This runs Level 1 spec checks (markdown lint, stub detection, change package completeness) before every push.
