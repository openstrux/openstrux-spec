# Change Packages

Every non-trivial spec change starts as a bounded change package.

## Structure

changes/
YYYY-NNN-short-title/
proposal.md
design.md
tasks.md
spec-delta/
fixtures/
benchmark-impact.md
acceptance.md

text

## Rules

- Every change package MUST identify affected manifesto objectives.
- Every change package MUST identify affected benchmark cases.
- Every accepted change MUST be merged back into stable spec modules.
- After merge, the package MAY be archived but MUST remain traceable.
