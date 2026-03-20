# Syntax Reference Review — Synthesis

**Date**: 2026-03-20
**Reviewed by**: DeepSeek, Gemini 3 Pro, GLM 4.6, GPT-5.4 (thinking), Opus 4.6 (thinking), Sonnet 4.6 (thinking)
**Document**: `specs/core/syntax-reference.md` (Openstrux 0.6.0)

---

## 1. Code Verification

I verified each LLM's generated `.strux` against the current syntax reference.
Errors fall into three categories:

### Correct patterns (all 6 LLMs got right)

- Basic ETL chain: `read-data → filter → pseudonymize → write-data`
- `from: f.reject` for filter DLQ
- `from: [users, orders]` for multi-input join
- `group → aggregate` implicit chain
- Context inheritance: `@source production`, `@target analytics`, panel override
- `@name { field: override }` for target override
- `policy("name")` in `@access` scope

### Common errors (spec-valid but wrong inference)

| Pattern | Error | Who | Verdict |
|---------|-------|-----|---------|
| `validate { schema: @type User }` | Schema value type not defined | DeepSeek | **Spec gap** — `cfg.schema` type is undefined |
| `validate { schema: "RequestSchema" }` | String schema ref | Gemini, Sonnet | **Spec gap** — same |
| `validate { schema: policy("request_schema") }` | Borrowed policy() | GLM | **Wrong** — policy() is for access policies, not schemas |
| `validate { schema: @order-schema }` | Named ref | Opus | **Guessed** — no named schema mechanism exists |
| `validate { schema: ApiPayload }` | Type name ref | GPT-5.4 | **Plausible** but unconfirmed |
| `window { size: "5m" }` | Duration format | All 6 | **Spec gap** — no duration literal grammar |
| `window { slide: "1m" }` | Slide parameter | Sonnet | **Guessed** — `slide` not listed as cfg/arg |
| `stream.kafka { bootstrap_servers: ... }` | Kafka fields | DeepSeek, GLM | **Spec gap** — no stream config fields defined |
| `stream.kafka { brokers: ... }` | Kafka fields | GPT-5.4, Opus, Sonnet | **Spec gap** — same, different field name guessed |
| `@ops { fallback: @backup-source }` | Fallback value | Opus, GPT-5.4 | **Spec gap** — fallback type undefined |
| `@ops { fallback: "default" }` | Fallback string | Gemini | **Spec gap** — same |
| `@ops { fallback: fn: mod/fallback.noop }` | Fallback fn ref | Sonnet | **Spec gap** — same |
| `@ops { circuit_breaker: { threshold: 5 } }` | CB sub-fields | GLM, Opus, Sonnet | **Spec gap** — circuit_breaker type undefined |
| `mode: "query", predicate: "SELECT ..."` | SQL in predicate | GLM | **Plausible** — `query` mode exists, but table/query selection is unspecified |
| Rod-level `@ops` placement | Before rod? Inside rod? | DeepSeek, GLM, Opus | **Spec gap** — rod-level decorator syntax undefined |
| `from: sink.failure` | Wiring failure knot | Gemini, GPT-5.4 | **Correct** — follows same pattern as reject |
| `from: output.reject` | DLQ for write-data reject | All 6 | **Correct** — spec explicitly documents this |

### Structural errors in generated code

| Pattern | Error | Who |
|---------|-------|-----|
| `write-data { target: ... }` (no rod name) | Missing `name =` before rod type | Gemini (cases a,e) |
| `split = split { ... }` | Rod name shadows rod type | GLM (case d) |
| `@ops` block between rods as a floating decorator | Ambiguous placement | GLM, GPT-5.4 (case f) |
| `orders = read-data { ... }` as 2nd rod without `from:` | Implicit chain reads previous rod's output, but read-data has no `in` | All join cases — works by convention but **rule unstated** |

---

## 2. Per-LLM Synthesis

### DeepSeek

- **Coverage**: 70-80%. Strongest on basic ETL and context. Weakest on streaming and @ops.
- **Code quality**: Correct structure throughout. Missed no shorthand rules.
- **Key insight**: Identified that `split` has no default output, breaking implicit chain — **valid and actionable**.
- **Blind spot**: Did not notice the `=` is still required between rod name and type.
- **Verdict**: "Needs editing pass" — agrees with consensus.

### Gemini 3 Pro

- **Coverage**: ~85% (most generous self-assessment). Clean, minimal code.
- **Code quality**: Omitted rod names in cases a and e (`write-data { ... }` instead of `name = write-data { ... }`). This is a **spec conformance error** — the shorthand rules say `name = rod-type { ... }`.
- **Key insight**: "Default Knots table is the hero" — correctly identifies the highest-value section. Also flagged knot path ambiguity (`f.reject` vs `f.out.reject`) — **valid and actionable**.
- **Blind spot**: Brevity sacrificed correctness (missing rod names).
- **Verdict**: "95% there, minor editing pass" — most optimistic, but own code has errors.

### GLM 4.6

- **Coverage**: ~70%. Most verbose code, most comments.
- **Code quality**: Generally correct. Used `split = split { ... }` (name shadows type — likely valid but poor practice). Used `policy("request_schema")` for validate schema — **wrong** (policy() is for access). Placed `@ops` as a floating block between rods — syntax undefined.
- **Key insight**: Identified predicate syntax ambiguity for read-data. Noted contradictions in shorthand vs verbose examples. But also reported false contradictions (e.g., "some examples show rods without `from:` reading from non-default outputs" — this is not in the doc).
- **Blind spot**: Several self-assessment claims are inaccurate (e.g., claiming guard inline OPA syntax is confident when it's genuinely ambiguous).
- **Verdict**: "Close, needs editing pass" — reasonable, but own review quality is lowest.

### GPT-5.4 (thinking)

- **Coverage**: 17-33% (harshest assessment). Most critical review.
- **Code quality**: Excellent. Most complete panels with full `@dp`, `@access` verbose forms. Only LLM to define a custom `@type ApiPayload` for validate.
- **Key insight**: "No syntax for selecting database tables" — **valid and important**. The spec never explains how to target a specific table in read-data or write-data. Also identified the strongest contradiction: `write-data.reject` is listed under `err` but described as "not a failure channel."
- **Blind spot**: Coverage assessment is too harsh — counts table-selection gap as blocking for cases where it's an adapter-level detail. Also counted `adc` as "not explained" but the document does explain it now.
- **Verdict**: "Needs another editing pass" — agrees with consensus but frames it more severely.

### Opus 4.6 (thinking)

- **Coverage**: ~75%. Most detailed and technically precise review.
- **Code quality**: Excellent. Cleanest shorthand. Only LLM to show `from: f.match` to explicitly resume chain after a branch — **correct and insightful**.
- **Key insight**: Best analysis of implicit chain ambiguity in branching graphs. Identified `window.out.windowed` → `aggregate.in.grouped` type mismatch. Identified `from:` namespace flattening as an unstated rule. Precisely estimated "200-300 additional tokens" to fix.
- **Blind spot**: None significant.
- **Verdict**: "One more focused editing pass" — most constructive framing with specific token budget.

### Sonnet 4.6 (thinking)

- **Coverage**: 4/6 fully or mostly covered. Balanced assessment.
- **Code quality**: Good. Only LLM to add `slide: "1m"` to window — not in spec, but reveals a real gap (sliding windows need a slide parameter). Used `group_id` for Kafka — plausible field name.
- **Key insight**: "Second read-data implicit chain rule unstated" — precisely identifies that source rods (in: —) must be exempt from implicit chaining, but no rule says so. Also identified `DataTarget` as the highest-impact missing type — **valid**, since write-data appears in every pipeline.
- **Blind spot**: Verbose example token savings estimate (280 tokens) is aggressive — the verbose form serves as the normalization reference.
- **Verdict**: "Not quite ready, one focused editing pass" — agrees with consensus.

---

## 3. Consensus Issues (ranked by frequency × impact)

Issues are ranked by how many LLMs flagged them AND how much they block correct generation.

### Tier 1 — Blocks generation (all or nearly all LLMs flagged)

| # | Issue | Flagged by | Impact |
|---|-------|-----------|--------|
| **1** | **@ops sub-field types undefined** (`retry`, `fallback`, `circuit_breaker`, `rate_limit`, `timeout`) — all 6 LLMs guessed different shapes | All 6 | Every resilience pattern fails. 6 different fallback syntaxes generated. |
| **2** | **Stream source config fields missing** (`stream.kafka`, `stream.pubsub`, `stream.kinesis`) — no config fields listed | All 6 | Streaming use case (d) blocked. 2 different field names guessed for Kafka alone (`brokers` vs `bootstrap_servers`). |
| **3** | **`validate` schema type undefined** — `cfg.schema` listed but value type unknown | All 6 | 5 different schema value syntaxes generated. API service use case (c) blocked. |
| **4** | **`DataTarget` type undefined** — write-data uses it but only DataSource has a union tree | Sonnet, GPT-5.4, Opus | Every pipeline writes data. Currently inferred by mirroring DataSource. |
| **5** | **Window duration format undefined** — `cfg.size` exists but format unknown | All 6 | All used `"5m"` by convention, but it's a guess. |

### Tier 2 — Causes ambiguity (3+ LLMs flagged)

| # | Issue | Flagged by | Impact |
|---|-------|-----------|--------|
| **6** | **Implicit chain breaks for branching graphs** — "previous rod" undefined when branches intervene | Opus, Sonnet, DeepSeek, GPT-5.4 | Incorrect wiring in any panel with DLQ + main chain. |
| **7** | **Rod-level `@ops` syntax undefined** — can you attach @ops to a rod? where? | DeepSeek, GLM, Opus, Sonnet | Resilience per-rod is the main use case for @ops. |
| **8** | **`from:` knot namespace flattening unstated** — `rod.reject` works but the rule that `from:` resolves across `out`/`err` is never stated | Opus, Gemini, Sonnet | Correct by example but not by rule. |
| **9** | **Source rods exempt from implicit chain** — second `read-data` in a panel has `in: —` but rule 2 says "reads previous rod's default output" | Sonnet, GPT-5.4 | Every multi-source panel depends on this unstated exception. |
| **10** | **`split` breaks implicit chain** — no default output, but rule 2 assumes one exists | DeepSeek, Gemini | Any panel with split routes must use explicit `from:`. |

### Tier 3 — Minor gaps (1-2 LLMs flagged, lower impact)

| # | Issue | Flagged by |
|---|-------|-----------|
| 11 | `guard` inline policy syntax ambiguous (`policy: opa: data.authz.allow` reads as double-colon) | Sonnet, Opus |
| 12 | Table/query selection in `read-data` unspecified (how to target `users` vs `orders` table) | GPT-5.4, GLM |
| 13 | `window → aggregate` type compatibility (`windowed` ≠ `grouped`) | Opus |
| 14 | `call.response` → `write-data.rows` type compatibility undefined | Opus, Sonnet |
| 15 | `@sec` and `@cert` sub-field types undefined | Opus, Sonnet |
| 16 | `StateBackend`, `PolicyRef`, `SchemaRef` type names dangling | Opus, Gemini |
| 17 | `COLLECT`, `CASE/WHEN/THEN/ELSE/END` reserved but unused | Opus, Sonnet |
| 18 | `pseudonymize.algo` valid values not enumerated | Sonnet |
| 19 | `encrypt.key_ref` format undefined | Sonnet |
| 20 | `guard.out.modified` never explained | Sonnet |

---

## 4. Suggested Improvements

Ordered by impact/token ratio. Token estimates are rough.

### Pass 1 — Close the generation blockers (~250 tokens)

**1. Add `@ops` field types** (~80 tokens)

```
@ops fields:
  retry: number                                // max attempts
  timeout: string                              // duration ("30s", "5m")
  fallback: @name | rod-name                   // named source or rod to execute on exhaustion
  circuit_breaker: { threshold: number, window: string }
  rate_limit: { max: number, window: string }
```

Place after the Decorators section. This closes issue #1 and #7 simultaneously.

**2. Add `DataTarget` note + stream config fields** (~80 tokens)

After the DataSource Union Tree section:

```
DataTarget follows the same union tree as DataSource.
Config patterns:
  stream.kafka { brokers, topic, credentials }
  stream.pubsub { project, topic }
  stream.kinesis { region, stream_name, credentials }
```

Closes issues #2 and #4.

**3. Define `validate.schema` type** (~20 tokens)

In the Compliance rod table or after it:

```
SchemaRef: @type reference or named schema from context. E.g., `schema: UserPayload`.
```

Closes issue #3.

**4. Define duration format** (~15 tokens)

In Grammar Essentials / Lexical:

```
duration      = number ("s" | "m" | "h" | "d")               // "30s", "5m", "24h"
```

Closes issue #5.

### Pass 2 — Close the ambiguities (~150 tokens)

**5. Clarify implicit chain in branching graphs** (~50 tokens)

Add to Implicit Chaining Rules, after rule 6:

```
7. Rods whose `in` is — (source rods: read-data, receive) always start a new
   chain regardless of position. They do not consume the previous rod's output.
8. A rod with explicit `from:` does not advance the implicit chain. The next
   rod without `from:` chains from the last rod that WAS part of the implicit
   chain, not from the branch rod.
```

Closes issues #6, #9, #10.

**6. State `from:` namespace rule** (~20 tokens)

Add to shorthand rules or implicit chaining:

```
`from: rod.knot` resolves the knot name across out and err namespaces.
Knot names are unique per rod; the namespace prefix is dropped in shorthand.
```

Closes issue #8.

**7. Clarify `split` has no default output** (~15 tokens)

In Default Knots table, change split row annotation:

```
| split | (named routes — no default, explicit `from:` required downstream) | data |
```

Closes issue #10.

**8. Show rod-level `@ops`** (~30 tokens)

One example in the shorthand equivalence block or error propagation:

```
src = read-data { source: @production, mode: "scan", @ops { retry: 5, fallback: @backup } }
```

Closes issue #7.

### Pass 3 — Nice to have (~100 tokens)

**9. `guard` policy shorthand example** — show `policy: opa: data.authz.allow` in a complete rod line.

**10. `window → group → aggregate` chain note** — one sentence: "`window.out.windowed` is compatible with `group.in.data` and `aggregate.in.grouped` via implicit chain."

**11. Table/query selection** — one sentence: "Table or collection selection is part of `arg.predicate` (e.g., `predicate: sql: SELECT * FROM users`)."

**12. Dangling types** — add one-line scope notes for `StateBackend`, `PolicyRef`, `COLLECT`, `CASE`.

---

## 5. Overall Verdict

| LLM | Self-assessed coverage | Verdict |
|-----|----------------------|---------|
| DeepSeek | 70-80% | Needs editing pass |
| Gemini 3 Pro | ~85% | 95% there, minor pass |
| GLM 4.6 | ~70% | Close, needs pass |
| GPT-5.4 | 17-33% | Needs editing pass |
| Opus 4.6 | ~75% | One focused pass, ~200-300 tokens |
| Sonnet 4.6 | 4/6 covered | One focused pass |

**Consensus**: The document is strong for linear ETL and context inheritance
(use cases a, b, e). It needs one editing pass for streaming, resilience,
and service patterns (use cases c, d, f). The gap is ~400 tokens of
additive content — no architectural changes needed.

**Reliability ranking** (by review quality, code correctness, and insight):
1. Opus 4.6 — most precise, best code, best insight on branching chains
2. Sonnet 4.6 — best structural analysis, identified DataTarget as #1 issue
3. GPT-5.4 — most critical, best at finding missing grammar, harshest but fair
4. DeepSeek — solid, identified split default output gap
5. Gemini 3 Pro — good insights but own code had errors (missing rod names)
6. GLM 4.6 — weakest review accuracy, some false findings
