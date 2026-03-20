# Openstrux 0.6.0 — Syntax Reference Review

**Reviewer**: Claude (Opus 4.6)  
**Date**: 2026-03-20  
**Document under review**: `syntax-reference.md` (Openstrux 0.6.0, 471 lines)

---

## 1. Generation Test — Can I Use This?

### 1a. Basic ETL Pipeline

Read users from Postgres, filter by country, pseudonymize PII, write to BigQuery, DLQ for rejected writes.

```strux
@panel user-etl {
  @dp { record: "RPA-2026-010" }
  @access { purpose: "data_migration", operation: "read" }
  src = read-data { source: @production, mode: "scan", predicate: deleted_at IS NULL }
  f = filter { predicate: country IN ("ES", "FR", "DE", "IT") }
  p = pseudonymize { algo: "sha256", fields: ["full_name", "email", "phone"] }
  sink = write-data { target: @analytics { dataset: "eu_users" } }
  dlq = write-data { target: @dlq, from: sink.reject }
}
```

**Self-assessment**:

- `read-data`, `filter`, `pseudonymize`, `write-data` chain: **Confident** — covered by the full panel example.
- DLQ wiring via `from: sink.reject`: **Confident** — the example shows `from: f.reject` for filter; `write-data` documents `reject` as an err knot. The shorthand `sink.reject` (without `.err.` prefix) is **Inferred** — the example uses `f.reject` not `f.err.reject`, but the rod table lists `reject` under `err`. The document's own example implies the shorthand drops the `err` segment, but this is never stated as a rule.
- `predicate` on `read-data` as an arg: **Confident** — listed in the rod table.

### 1b. Multi-Source Join

Two Postgres tables, join on user_id, aggregate order totals by country, write to BigQuery.

```strux
@panel orders-by-country {
  @dp { record: "RPA-2026-011" }
  @access { purpose: "reporting", operation: "read" }
  users = read-data { source: @production, mode: "scan", fields: [id, address.country AS country] }
  orders = read-data { source: @production, mode: "scan", fields: [user_id, amount] }
  j = join { mode: "inner", on: left.user_id == right.id, from: [users, orders] }
  g = group { key: country }
  agg = aggregate { fn: [SUM(amount) AS total_amount, COUNT(*) AS order_count] }
  sink = write-data { target: @analytics { dataset: "order_summary" } }
}
```

**Self-assessment**:

- `join` with `from: [users, orders]`: **Confident** — multi-input normalization example covers this exactly.
- `group` → `aggregate` implicit chain: **Confident** — default knots table shows `group.out.grouped` → `aggregate.in.grouped`.
- `fields` on `read-data` as projection: **Confident** — listed as an arg.
- Whether `from: [users, orders]` means first=left, second=right: **Confident** — rule 4 of implicit chaining says exactly this.

### 1c. API-Triggered Service

HTTP POST → validate → guard (OPA) → call external API → write response to DB.

```strux
@panel api-handler {
  @dp { record: "RPA-2026-012" }
  @access { purpose: "service_request", operation: "write" }
  req = receive { trigger: http { method: "POST", path: "/api/orders" } }
  v = validate { schema: @order-schema }
  auth = guard { policy: opa: data.authz.allow }
  ext = call {
    target: http { base_url: env("EXT_API_URL"), tls: true },
    method: "post", path: "/process"
  }
  sink = write-data { target: @primary-db }
}
```

**Self-assessment**:

- `receive` with `http` trigger: **Confident** — documented.
- `validate` with `schema`: **Confident** — listed in rod table as `cfg.schema`.
- `guard` with OPA policy shorthand: **Inferred** — the guard rod table says `arg.policy (shorthand)` and guard policy expressions show `opa: data.authz.allow`. But the exact shorthand form `policy: opa: data.authz.allow` (a prefixed expression inside a shorthand cfg field) is not shown in a complete example. I'm reasonably confident but uncertain whether the `opa:` prefix works inline or needs wrapping.
- `@order-schema` for validate: **Guessed** — there is no example of `validate` with a named reference. The document says `cfg.schema` exists but doesn't show what values it accepts. Is it a type path? A policy reference? A named schema from context? Unknown.
- `call` → `write-data` implicit chain: **Inferred** — `call` default out is `response`, `write-data` default in is `rows/elements`. Whether `response` auto-maps to `rows` is unclear; this might need explicit `from:`.

### 1d. Streaming Pipeline with Windowing

Read from Kafka, sliding window, aggregate counts per window, split by region.

```strux
@panel streaming-counts {
  @dp { record: "RPA-2026-013" }
  @access { purpose: "analytics", operation: "read" }
  src = read-data {
    source: stream.kafka { brokers: env("KAFKA_BROKERS"), topic: "events", credentials: secret_ref { provider: vault, path: "kafka/creds" } },
    mode: "stream"
  }
  w = window { kind: "sliding", size: "5m" }
  agg = aggregate { fn: COUNT(*) AS event_count }
  s = split { routes: { eu: region IN ("EU-WEST", "EU-EAST"), us: region == "US", apac: region == "APAC", other: * } }
  eu-sink = write-data { target: @eu-target, from: s.eu }
  us-sink = write-data { target: @us-target, from: s.us }
  apac-sink = write-data { target: @apac-target, from: s.apac }
}
```

**Self-assessment**:

- `window` rod with `kind` and `size`: **Confident** — rod table lists `cfg.kind` and `cfg.size`.
- `split` with named routes and `from: s.eu`: **Confident** — split routes syntax is documented, and the default out table says "(named routes)".
- Kafka source inline config: **Inferred** — the DataSource union tree mentions `stream (kafka, ...)` but the config fields for Kafka (`brokers`, `topic`) are not documented. I used `brokers` and `topic` by convention. Only Postgres has a spelled-out config pattern.
- `size: "5m"` as a duration: **Guessed** — the document says `cfg.size` exists but gives no format or example for duration/window-size values.
- `window` → `aggregate` chain: **Inferred** — `window.out.windowed` should chain into `aggregate.in.grouped`? Actually the default-in for aggregate is `grouped`, not `windowed`. This is a potential type mismatch. The document doesn't clarify whether `windowed` is compatible with `grouped`.

### 1e. Context-Heavy Project

`strux.context` with shared Postgres, BigQuery, default @dp, default @access. Two panels inheriting.

```strux
// strux.context
@context {
  @dp { controller: "Acme Corp", controller_id: "ES-B87654321", dpo: "dpo@acme.com" }
  @access { intent: { basis: "legitimate_interest" }, scope: policy("data-team-read") }
  @source production = db.sql.postgres {
    host: env("DB_HOST"), port: 5432, db_name: "main", tls: true,
    credentials: secret_ref { provider: gcp_secret_manager, path: "projects/acme/secrets/pg" }
  }
  @target analytics = db.sql.bigquery {
    project: "acme-analytics", location: "EU",
    credentials: adc {}
  }
  @target dlq = stream.pubsub { project: "acme", topic: "dlq" }
  @ops { retry: 3, timeout: "30s" }
}
```

```strux
// domain-a/pipelines/user-export.strux
@panel user-export {
  @dp { record: "RPA-2026-020" }
  @access { purpose: "data_export", operation: "read" }
  db = read-data { source: @production, mode: "scan" }
  f = filter { predicate: status == "active" }
  sink = write-data { target: @analytics { dataset: "active_users" } }
}
```

```strux
// domain-a/pipelines/order-summary.strux
@panel order-summary {
  @dp { record: "RPA-2026-021" }
  @access { purpose: "reporting", operation: "read" }
  db = read-data { source: @production, mode: "query", predicate: sql: created_at > CURRENT_DATE - INTERVAL '30 days' }
  g = group { key: product_category }
  agg = aggregate { fn: [SUM(amount) AS total, COUNT(*) AS count] }
  sink = write-data { target: @analytics { dataset: "order_summary" } }
}
```

**Self-assessment**:

- `strux.context` syntax: **Confident** — documented with example.
- Named `@source` and `@target` with `@name` resolution: **Confident** — documented.
- `@target @name { override }` for inline field override: **Confident** — documented in Built-in References.
- Inheriting `@dp` fields (controller, controller_id, dpo) while overriding `record`: **Confident** — field-level merge, panel wins.
- Inheriting `@access` basis while providing panel-specific purpose: **Inferred** — the document says @access is field-level narrowing, but the merge semantics between `intent` sub-fields (basis from context, purpose from panel) is not fully spelled out. Can you provide `purpose` at panel level and `basis` at context level and have them merge into one `intent`? The shorthand flattening rule (rule 4) says flat `@access { purpose, operation }` routes to `intent` sub-structure, but it's unclear if this merge works across context + panel boundaries.
- `@ops` inheritance: **Confident** — table says "Field-level merge, nearest wins".

### 1f. Error Handling and Resilience

Retry + fallback on read-data, filter rejects to DLQ, write-data rejects to error topic. `@ops` decorators.

```strux
@panel resilient-pipeline {
  @dp { record: "RPA-2026-030" }
  @access { purpose: "ingestion", operation: "read" }
  @ops { retry: 5, timeout: "60s", circuit_breaker: { threshold: 3, window: "120s" } }

  src = read-data {
    source: @production, mode: "scan",
    @ops { fallback: @backup-source }
  }
  f = filter { predicate: quality_score > 0.5 }
  reject-sink = write-data { target: @dlq, from: f.reject }
  t = transform { mode: "map", fields: [id, name, score], from: f.match }
  sink = write-data { target: @analytics }
  err-sink = write-data { target: stream.kafka { brokers: env("KAFKA_BROKERS"), topic: "write-errors" }, from: sink.reject }
}
```

**Self-assessment**:

- `@ops` at panel level with `retry`, `timeout`, `circuit_breaker`: **Confident** — decorator fields are listed.
- Rod-level `@ops { fallback: ... }`: **Guessed** — the document mentions `fallback?` as an `@ops` field and the error propagation rules say fallback runs after retries exhaust. But the syntax for specifying a fallback (what is the value type? a named source? a rod reference? a lambda?) is **completely undocumented**.
- `circuit_breaker` record shape (`threshold`, `window`): **Guessed** — listed as a field name but no type, no sub-fields, no example.
- `from: f.match` to explicitly wire the default output: **Inferred** — needed because implicit chain would follow `reject-sink`, not `f`. The document doesn't show this "skip back" pattern but `from:` overrides implicit chain per rule 3.
- Wiring `sink.reject` for write-data rejects: **Confident** — follows the DLQ pattern and error propagation docs.

---

## 2. Coverage Test — Does It Cover 90%?

| Use Case                  | Fully Covered? | Gaps                                                                                                                                               |
| ------------------------- | -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1a. Basic ETL             | **Yes**        | Minor: err-knot shorthand (`f.reject` vs `f.err.reject`) is example-implied, not stated as a rule                                                  |
| 1b. Multi-source join     | **Yes**        | Clean coverage                                                                                                                                     |
| 1c. API-triggered service | **Mostly**     | `validate` cfg.schema value types unknown; `call` → `write-data` type compatibility unclear                                                        |
| 1d. Streaming + windowing | **Partially**  | Kafka config fields undocumented; window size format undocumented; `window` → `aggregate` type compatibility unclear                               |
| 1e. Context inheritance   | **Mostly**     | @access intent field merging across context/panel boundary underspecified                                                                          |
| 1f. Error handling        | **Partially**  | `fallback` value type undocumented; `circuit_breaker` sub-fields undocumented; `rate_limit` undocumented; rod-level `@ops` override syntax unclear |

### Specific coverage questions:

**Were shorthand rules clear enough to apply without ambiguity?**
Mostly yes. The four shorthand rules are concise and the verbose↔shorthand equivalence example is excellent. One gap: the shorthand for error knot references (is it `rod.reject` or `rod.err.reject`?) is shown by example but never stated as a rule.

**Were default knots and implicit chaining explicit enough?**
Good for linear chains. Problematic for branching graphs — when a rod consumes a non-default output (e.g., `reject-sink` reads `f.reject`), the implicit chain for the _next_ rod is broken. Rule 2 says "reads the previous rod's default output" but "previous" could mean declaration-order-previous (which is `reject-sink`) or chain-previous (which is `f`). The multi-input example helps, but a branching example is missing.

**Were decorators explained enough to use correctly?**
`@dp` and `@access`: Yes. `@ops`: Partially — field _names_ are listed but field _types_ and _values_ are not (what is `retry`'s type? a number? a record with `max`, `backoff`?). `@sec`: Barely — three field names, no types, no example. `@cert`: Minimal — one example line, no explanation of `hash` or `version` semantics.

**Were credential and reference forms clear?**
`env()`, `secret_ref{}`, `@name`, `@name { override }`, `adc{}`: all clear and exemplified. `policy("name")`: clear for `@access` scope, unclear when used elsewhere (e.g., `guard` vs. `@access`).

---

## 3. Clarity and Completeness Audit

### 3a. Ambiguities

1. **`from:` shorthand — does it include the knot namespace?**
   The full panel example shows `from: f.reject` for a filter's error output. The verbose form is `snap f.out.reject -> in.elements` (or is it `snap f.err.reject`?). The shorthand drops `out.` / `err.` — but is `f.reject` sugar for `f.out.reject` or `f.err.reject`? For filter, `reject` is listed under `out` (not `err`), so `f.reject` = `f.out.reject`. But for `write-data`, `reject` is listed under `err`. Is `sink.reject` then `sink.err.reject`? The shorthand appears to flatten the namespace, but this rule is never stated.
   **Likely intent**: `from:` resolves the knot name regardless of namespace (`out` or `err`), since knot names are unique per rod.

2. **"Previous rod" in implicit chaining when branches exist.**
   Rule 2: "each subsequent rod without `from:` reads the previous rod's default output." Does "previous" mean lexically previous in source order, or the last rod in the current chain branch? Consider:

   ```
   f = filter { ... }
   dlq = write-data { target: @dlq, from: f.reject }
   t = transform { ... }   // implicit chain — does this read dlq.receipt or f.match?
   ```

   **Likely intent**: "previous" means the last rod that is part of the main chain (i.e., `f`), skipping branches. But this requires tracking chain membership, which the document doesn't specify.
   **Alternative reading**: "previous" is strictly lexical order (the rod declared immediately before), meaning `t` reads `dlq.receipt`. This would be a surprising default.

3. **`@ops` at rod level vs. panel level.**
   The context example shows panel-level `@ops`. The error propagation rules reference `@ops { retry }` and `@ops { fallback }` as per-rod behaviors. But can you attach `@ops` to an individual rod? The inheritance table says "@ops: Field-level merge, nearest wins" — implying rod-level is possible. But the panel structure syntax doesn't show where rod-level decorators go syntactically (before the rod? inside the rod block?).
   **Likely intent**: Rod-level `@ops` is supported, probably inside the rod block. But the syntax is not shown.

4. **`window` output type vs. `aggregate` input type.**
   `window.out.windowed` is the default output. `aggregate.in.grouped` is the default input. These are different names. Does the implicit chain coerce `windowed` → `grouped`? Or does `window` → `aggregate` require an explicit `from:`?
   **Likely intent**: Implicit chain connects them by position regardless of knot name (it's about types, not names). But the document doesn't state this.

### 3b. Missing Grammar

1. **Duration/interval literals** — `timeout: "30s"`, `size: "5m"` appear in examples but there is no grammar rule for duration values. What are valid units? Is it always a string? Can you write `30s` without quotes?

2. **`cfg.circuit_breaker` record** — mentioned as an `@ops` field but never defined. Sub-fields are unknown.

3. **`cfg.rate_limit` record** — same as above.

4. **`cfg.fallback` value** — completely undefined. Is it a source reference, a rod reference, a literal, a type path?

5. **`@sec` sub-fields** — `encryption?`, `classification?`, `audit?` are listed with question marks (optional) but no type definitions. What values do they take?

6. **`validate`'s `cfg.schema`** — type undefined. Is it a `@type` reference? A JSON Schema reference? A type path?

7. **`store`'s `cfg.backend: StateBackend`** — `StateBackend` is never defined in the union tree or anywhere else.

8. **Inline type construction in rod config** — the example shows `db.sql.postgres { host: ..., port: ..., ... }` inline. The grammar essentials don't define this form (type path followed by `{ key: val }` record literal). It's shown by example but lacks a production rule.

9. **List literal syntax** — `["full_name", "email"]` appears in examples but the grammar section only defines `string`, `number`, `bool`. No array/list literal rule.

10. **`@when` and `@adapter`** — reserved but described only as "defined in deeper spec modules". For a self-sufficient reference, these should either be briefly explained or explicitly marked as out of scope with a note not to use them.

### 3c. Contradictions

1. **`filter` outputs: `out` vs. `err`?** — The rod table for `filter` lists outputs as `out.match + out.reject` (both under `out`). But the error propagation section discusses `err` knots. If `reject` is under `out` (not `err`), then filter's rejected data is not an "error" and doesn't trigger retry/fallback. This is consistent but potentially confusing, because `write-data`'s `reject` _is_ listed under `err`. The document should clarify: filter's `reject` is a data branch (under `out`), while write-data's `reject` is an error branch (under `err`). The distinction matters for `@ops` cascade behavior.

2. **Minor: token count claim.** The shorthand example claims "~142 tokens" but this depends entirely on the tokenizer. Not a contradiction per se, but it could mislead if someone counts differently.

### 3d. Dangling References

1. **`StateBackend`** — used in `store` rod definition, never defined. No union tree, no type path, no config fields.
2. **`ReadMode` values** — listed (`scan`, `lookup`, `multi_lookup`, `query`, `stream`) but semantic differences not explained. When do you use `query` vs. `scan`? What does `multi_lookup` accept that `lookup` doesn't?
3. **`PolicyRef`** — used in `guard` (`cfg.policy: PolicyRef`) but the type is never defined. Is it the same as `policy("name")`? A broader union?
4. **`@when`** — reserved keyword, no definition, no scope note.
5. **`@adapter`** — same as `@when`.
6. **`Trigger` sub-types** — `event{source,topic}`, `schedule{cron?,interval?}`, `queue{source,queue}` are listed but their config fields are unexplained. What is `event.source`? A type path? A string?
7. **`COLLECT` aggregate** — reserved keyword but not shown in the aggregation expressions section. No example, no explanation.
8. **`CASE WHEN THEN ELSE END`** — reserved keywords for case expressions, but no expression example shows their use.

### 3e. Edge Cases

**1. A rod with both `retry` and `fallback` in `@ops` — which runs first?**

✅ **Covered.** Error propagation rule 2 states: "apply `@ops { retry }` first (re-execute the rod), then `@ops { fallback }` if retries are exhausted." Clear ordering.

**2. `write-data` produces both `failure` and `reject` — how do they differ and how do you wire each?**

✅ **Covered.** Error propagation section explicitly addresses this: `failure` = transport/system error (follows retry/fallback cascade), `reject` = data-level rejection (typed branch, wire like any non-default output, typically to DLQ). Well explained.

**3. A `join` rod in implicit chain position — where does the second input come from?**

⚠️ **Partially covered.** Implicit chaining rule 4 says `from: [rod1, rod2]` for multi-input rods. But the document doesn't say what happens if you write `j = join { ... }` without `from:` in implicit chain position. Presumably it's an error (join requires two inputs, implicit chain provides one). This should be stated explicitly: **multi-input rods require explicit `from:`; implicit chain cannot satisfy them.**

**4. A `filter` followed by two downstream rods — one consuming `match`, one consuming `reject` — how is the implicit chain resolved?**

⚠️ **Underspecified.** Rule 5 says: "Implicit chain follows the default output only (match, not reject). Non-default outputs require explicit `from:`." This tells us the `reject` consumer needs `from: f.reject`. But what about the rod _after_ the reject consumer — does implicit chain resume from `f.match` or from the reject consumer? This is the "previous rod" ambiguity from §3a.2.

**5. A panel with no `@access` and no inherited access context — what happens?**

✅ **Covered.** Semantic essentials: "Empty AccessContext → deny (fail-closed)." Clear and definitive.

### 3f. Token Efficiency

The document is generally well-optimized for token budget. Specific observations:

1. **Redundant**: The verbose full panel example (lines 410–437) and the shorthand version (lines 443–451) are both valuable, but the verbose form repeats information already conveyed by the rod tables and shorthand rules. Consider whether the verbose form could be reduced to a diff annotation showing what shorthand removes, rather than a full duplicate.

2. **Redundant**: The shorthand rules are stated in prose (lines 41–48) and then re-stated as normalization rules in the grammar section (lines 333–338). These could be consolidated.

3. **Verbose**: The "Specification Map" table at the end (lines 458–470) adds ~80 tokens for links an LLM cannot follow. Since the document's purpose is self-sufficiency, this table is primarily useful as a signal that "if you need more, look here" — but the LLM won't load those files during generation. Could be reduced to a single line: "Deeper specs: grammar.md, semantics.md, type-system.md, expression-shorthand.md, panel-shorthand.md, config-inheritance.md, conformance.md, ir.md, locks.md."

4. **Efficient**: The rod tables are excellent — dense information in minimal tokens. The expression shorthand section is also very token-efficient.

5. **Structural overhead**: Markdown `---` horizontal rules add negligible tokens but serve readability. Worth keeping.

---

## 4. Summary

### Generation Readiness

**Mostly.** An LLM can generate correct `.strux` for linear ETL pipelines, joins, context inheritance, and basic compliance flows with high confidence. It will struggle with streaming config details, `@ops` sub-field types, `validate` schema references, `@sec` details, and any branching graph where implicit chain resolution becomes ambiguous.

### Coverage

| Use Case                  | Verdict                                                                       |
| ------------------------- | ----------------------------------------------------------------------------- |
| 1a. Basic ETL             | ✅ Fully covered (no guessing)                                                |
| 1b. Multi-source join     | ✅ Fully covered (no guessing)                                                |
| 1c. API-triggered service | ⚠️ 80% covered (validate schema, call→write chain unclear)                    |
| 1d. Streaming + windowing | ⚠️ 60% covered (Kafka config, window size format, window→aggregate type gap)  |
| 1e. Context inheritance   | ⚠️ 85% covered (@access intent merging across boundaries)                     |
| 1f. Error handling        | ⚠️ 50% covered (fallback type, circuit_breaker fields, rod-level @ops syntax) |

**Overall: 4 of 6 use cases fully or mostly covered without guessing. ~75% coverage of the target 90%.**

### Top 5 Issues (ranked by impact on generation correctness)

1. **`@ops` sub-field types are undefined.** `retry`, `fallback`, `circuit_breaker`, `rate_limit`, `timeout` are listed as field names with no types, no sub-fields, no value examples. Since `@ops` is the primary resilience mechanism, this is a major gap. An LLM will have to guess the shape of every resilience configuration. **Fix: add a type definition block for each @ops field, even if brief (e.g., `retry: number | { max: number, backoff: "exponential" | "linear" }`).**

2. **Implicit chain resolution in branching graphs is ambiguous.** "Previous rod" is undefined when branches (DLQ sinks, reject handlers) intervene in declaration order. This is the single most likely source of incorrect wiring in generated `.strux`. **Fix: add one sentence clarifying that `from:` overrides implicit chain for that rod only, and the next rod without `from:` chains from the last rod in the main (default-output) chain, or require explicit `from:` after any branch.**

3. **DataSource/DataTarget config fields are documented only for Postgres.** Kafka, PubSub, Kinesis, MongoDB, DynamoDB, Firestore, MySQL, BigQuery (partially) all lack config field listings. An LLM generating a Kafka source will have to invent field names. **Fix: add a config-fields table per source/target type, even if only 3–4 key fields each.**

4. **`validate` schema, `store` backend, `guard` PolicyRef — value types are opaque.** Three rod types have config parameters whose types are named but never defined. An LLM cannot generate correct `validate`, `store`, or policy-driven `guard` configurations. **Fix: define `SchemaRef`, `StateBackend`, `PolicyRef` as union types, at least at the leaf level.**

5. **`from:` shorthand for error knots — rule is implicit.** The document shows `from: f.reject` by example (filter, under `out`) and `from: sink.reject` by inference (write-data, under `err`). The rule that `from: rod.knot` resolves the knot name across `out`/`err` namespaces is never stated. **Fix: add one sentence to the shorthand rules: "`from: rod.knot` resolves the knot name across all namespaces (out, err); knot names are unique per rod."**

### Verdict

**The document needs one more focused editing pass before it is self-sufficient for LLM generation.**

It is already excellent for its core competency: the linear ETL / compliance pipeline pattern that the full example demonstrates. The rod table, default knots table, expression shorthand, and implicit chaining rules are well-designed and token-efficient.

The gap is around the _edges_ of that core: resilience configuration, branching graph wiring, non-Postgres source configs, and several opaque type references. These are exactly the areas where an LLM will hallucinate, because it has enough structure to attempt a plausible answer but not enough constraint to get it right.

**Recommended priority for the next pass:**

1. Define `@ops` field types (biggest bang for token budget)
2. Add a branching-graph implicit chain example or rule
3. Add config field tables for at least Kafka, BigQuery, and PubSub
4. Define `SchemaRef`, `StateBackend`, `PolicyRef`
5. State the `from:` namespace-flattening rule explicitly

With these five additions (estimated ~200–300 additional tokens), the document would clear the 90% generation threshold comfortably.
