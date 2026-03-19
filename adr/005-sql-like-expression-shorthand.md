# ADR-005: SQL-Like Expression Shorthand

- **Status:** Accepted
- **Date:** 2026-03-19
- **Context:** Expressions (filter predicates, projections, aggregations,
  join conditions, sort orders, split routes, guard policies) need a source
  syntax. The source form is what humans and LLMs write; the compiled AST
  is what the compiler produces for pushdown analysis and audit.

## Decision

Use SQL-like syntax as the expression shorthand. Not because OpenStrux is
SQL-centric, but because:

1. **Every LLM knows SQL syntax** — zero learning curve for generation
2. **Every backend developer knows SQL** — zero learning curve for reading
3. **Most token-efficient predicate notation that exists** — ~5-15 tokens
   per predicate vs ~30-60 for AST form
4. **Maps cleanly to portable AND source-specific expressions**

The shorthand supports:

- **Portable expressions** (no prefix) — push to any source
- **Source-specific expressions** (prefixed: `sql:`, `mongo:`, `kafka:`) —
  push only to matching source
- **Function references** (`fn:`) — opaque, no pushdown
- **External policy references** (`opa:`, `cedar:`) — guard-only

### Two Forms, Same Semantics

| Form | Purpose | Who writes it | Token cost |
|------|---------|--------------|------------|
| Source (shorthand) | Authoring in .strux files | LLMs and humans | Low |
| Compiled (strux AST) | Machine processing, pushdown, audit | Compiler | High |

The AST (defined in `expressions.strux`) is never written by hand.

## Alternatives Considered

**Custom DSL:** Higher learning curve for both humans and LLMs. Would
require training data that doesn't exist. Every token spent teaching syntax
is a token not spent on the actual pipeline. Rejected.

**JSON-based expressions:** Verbose (~4-5x more tokens), hard to read,
error-prone for LLMs to generate correctly (bracket matching). Rejected.

**Native language expressions (Python/JS):** Breaks target independence.
A Python expression can't be pushed to SQL or emitted as TypeScript.
Rejected.

**No shorthand (write AST directly):** The AST form is ~4x more tokens
and unreadable. Defeats the "compact source" principle. Rejected.

## Consequences

- SQL familiarity is a prerequisite for advanced expression authoring
  (mitigated by universal SQL literacy among backend engineers and LLMs)
- Source-specific prefixes (`sql:`, `mongo:`) enable full power of the
  underlying system when needed, at the cost of portability
- Pushdown validation is a compile-time check: wrong prefix on wrong
  source type = compile error, not runtime surprise
- The compiler detects expression type from syntax (prefix or no prefix)
  and routes to the appropriate parser
