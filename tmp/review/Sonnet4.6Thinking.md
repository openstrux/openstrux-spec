I've read the full document. Let me now produce the complete review.

Openstrux 0.6.0 Syntax Reference — External Review

1. Generation Test
   Use Case a — Basic ETL Pipeline
   text
   @panel user-etl {
   @dp { controller: "Acme Corp", controller_id: "B-123",
   dpo: "dpo@acme.com", record: "RPA-2026-001" }
   @access { purpose: "geo_segmentation", basis: "legitimate_interest", operation: "read" }

db = read-data {
source: db.sql.postgres {
host: env("DB_HOST"), port: 5432, db_name: "production", tls: true,
credentials: secret_ref { provider: vault, path: "projects/acme/secrets/pg" }
},
mode: "scan"
}
f = filter { predicate: address.country IN ("US","CA","GB") AND deleted_at IS NULL }
p = pseudonymize { algo: "sha256", fields: ["email","full_name","national_id"] }
sink = write-data { target: db.sql.bigquery {
project: "acme-analytics", dataset: "users_eu", location: "EU", credentials: adc {}
}}
dlq = write-data { target: stream.pubsub { project: "acme", topic: "dlq" },
from: sink.reject }
}
Self-assessment:

Confident — read-data, filter, pseudonymize, write-data structure; implicit chain through p→sink; from: sink.reject (explicit in the shorthand full-panel example).

Inferred — adc {} as BigQuery credentials (shown in the verbose full example).

Guessed — stream.pubsub config fields (project, topic): the document names stream.pubsub as a valid type path but never shows its config shape.

Use Case b — Multi-Source Join
text
@panel orders-by-country {
@dp { controller: "Acme Corp", controller_id: "B-123",
dpo: "dpo@acme.com", record: "RPA-2026-002" }
@access { purpose: "financial_reporting", basis: "legitimate_interest", operation: "read" }

users = read-data { source: @production, mode: "scan" }
orders = read-data {
source: db.sql.postgres {
host: env("DB_HOST"), port: 5432, db_name: "orders", tls: true,
credentials: secret_ref { provider: vault, path: "projects/acme/secrets/pg-orders" }
},
mode: "scan"
}
j = join { mode: "inner", on: left.user_id == right.user_id, from: [users, orders] }
g = group { key: country }
agg = aggregate { fn: [SUM(amount) AS total_amount, COUNT(*) AS order_count] }
sink = write-data { target: @analytics { dataset: "orders_by_country" } }
}
Self-assessment:

Confident — join with from: [left, right] (exact example in Semantic Essentials); group/aggregate chaining; @name { override } syntax.

Inferred — orders (second read-data) silently breaks the implicit chain rule. Rule 2 says every subsequent rod reads the previous rod's default output, but read-data has in: —. The example in Semantic Essentials implies this is legal, but no rule states it explicitly.

Inferred — After join, country is referenced by group without a qualifying table prefix; cross-join field scoping after join is never described.

Use Case c — API-Triggered Service
text
@panel api-proxy {
@dp { controller: "Acme Corp", controller_id: "B-123",
dpo: "dpo@acme.com", record: "RPA-2026-003" }
@access { purpose: "api_proxy", basis: "legitimate_interest", operation: "read" }

recv = receive { trigger: http { method: "POST", path: "/v1/process" } }
v = validate { schema: "RequestSchema" }
g = guard { policy: opa: data.authz.allow }
api = call {
target: http { base_url: env("UPSTREAM_URL"), tls: true,
auth: secret_ref { provider: vault, path: "api/key" } },
method: "post"
}
db = write-data {
target: db.sql.postgres {
host: env("DB_HOST"), port: 5432, db_name: "responses", tls: true,
credentials: secret_ref { provider: vault, path: "projects/acme/secrets/pg" }
}
}
}
Self-assessment:

Confident — receive, call, validate presence; http trigger and target forms; opa: prefix for guard.

Inferred — guard inline policy expression syntax: the document lists arg.policy (shorthand) and shows opa: data.authz.allow as an expression, but never shows policy: opa: data.authz.allow as a complete assignment — the double-colon reads awkwardly.

Guessed — Type compatibility between call.response (a service response object) and write-data.in.rows/elements. The document never defines how a service response maps to a DataTarget input. A transform rod in between may be required, but this is not indicated.

Guessed — cfg.schema for validate accepts a string name. The document lists cfg.schema without defining what value it takes (inline schema? registry reference?).

Use Case d — Streaming Pipeline with Windowing
text
@panel kafka-windowed {
@dp { controller: "Acme Corp", controller_id: "B-123",
dpo: "dpo@acme.com", record: "RPA-2026-004" }
@access { purpose: "stream_analytics", basis: "legitimate_interest", operation: "read" }

src = read-data {
source: stream.kafka {
brokers: [env("KAFKA_BROKERS")], topic: "events",
group*id: "analytics-group",
credentials: secret_ref { provider: vault, path: "kafka/creds" }
},
mode: "stream"
}
w = window { kind: "sliding", size: "5m", slide: "1m" }
g = group { key: region }
agg = aggregate { fn: COUNT(*) AS event*count }
sp = split { routes: { eu: region IN ("ES","FR","DE"), us: region == "US", other: * } }
eu_sink = write-data { target: @analytics { dataset: "eu" }, from: sp.eu }
us_sink = write-data { target: @analytics { dataset: "us" }, from: sp.us }
other_sink = write-data { target: @analytics { dataset: "other" }, from: sp.other }
}
Self-assessment:

Confident — window rod type and knots; split routes and from: sp.route_name pattern; mode: "stream" on read-data.

Inferred — window → group → aggregate implicit chain makes semantic sense, but the document never explicitly describes how windowed batches flow into group/aggregate (e.g., whether grouping applies within each window or globally).

Guessed — Kafka source config fields (brokers, topic, group_id). The document names stream.kafka as a valid type path but shows zero config fields for any stream source.

Guessed — window.size and window.slide value format. cfg.size is listed but no duration string format ("5m", 300, {value:5,unit:"minutes"}) is ever shown.

Use Case e — Context-Heavy Project
strux.context:

text
@context {
@dp {
controller: "Acme Corp", controller_id: "B-123", dpo: "dpo@acme.com"
}
@access {
intent: { basis: "legitimate_interest" },
scope: policy("default-read")
}
@source production = db.sql.postgres {
host: env("DB_HOST"), port: 5432, db_name: "production", tls: true,
credentials: secret_ref { provider: gcp_secret_manager, path: "projects/acme/secrets/pg" }
}
@target analytics = db.sql.bigquery {
project: "acme-analytics", dataset: "default", location: "EU", credentials: adc {}
}
@ops { retry: 3, timeout: "30s" }
}
Panel 1 — user export:

text
@panel user-export {
@dp { record: "RPA-2026-005" }
@access { purpose: "user_reporting", operation: "read" }

db = read-data { source: @production, mode: "scan" }
f = filter { predicate: status == "active" }
p = pseudonymize { algo: "sha256", fields: ["email","full_name"] }
sink = write-data { target: @analytics { dataset: "active_users" } }
}
Panel 2 — order aggregation:

text
@panel order-summary {
@dp { record: "RPA-2026-006" }
@access { purpose: "financial_reporting", operation: "read" }

db = read-data { source: @production, mode: "scan",
fields: [order_id, user_id, amount, country, created_at] }
g = group { key: country }
agg = aggregate { fn: [SUM(amount) AS total, COUNT(*) AS orders] }
sink = write-data { target: @analytics { dataset: "order_totals" } }
}
Self-assessment:

Confident — strux.context structure; @source/@target named references; @name { override } syntax; field-level @dp and @ops merging described explicitly in the inheritance table.

Inferred — A panel @access block with only purpose and operation merges with context-level basis. The document says "field-level merge, panel wins on conflict" for @dp and @ops, but the access narrowing rule for individual intent fields (vs. the whole scope) is not explicitly illustrated.

Use Case f — Error Handling and Resilience
text
@panel resilient-sync {
@dp { controller: "Acme Corp", controller_id: "B-123",
dpo: "dpo@acme.com", record: "RPA-2026-007" }
@access { purpose: "data_sync", basis: "legitimate_interest", operation: "read" }
@ops { retry: 3, timeout: "30s", circuit_breaker: { threshold: 5, window: "60s" },
fallback: fn: mod/fallback.noop }

db = read-data { source: @production, mode: "scan" }
f = filter { predicate: is_valid == true }
sink = write-data { target: @analytics }

filter_dlq = write-data {
target: stream.pubsub { project: "acme", topic: "filter-rejects" },
from: f.reject
}
write_dlq = write-data {
target: stream.kafka { brokers: [env("KAFKA_BROKERS")], topic: "write-errors" },
from: sink.reject
}
}
Self-assessment:

Confident — Retry-before-fallback ordering (explicitly stated in Error Propagation); f.reject and sink.reject as DLQ wire targets; failure vs reject distinction.

Inferred — @ops at panel level (document shows it at context level in examples; inheritance table says it's inheritable, implying panel-level is also valid).

Guessed — circuit_breaker and fallback value shapes. Both listed as optional fields in @ops but no type definition or example is given.

Guessed — Stream target config fields (same gap as case d).

2. Coverage Test
   Did the document contain all syntax needed?
   Use Case Fully Covered? Key Gaps
   a. Basic ETL Yes — the full verbose example is this use case stream.pubsub config fields
   b. Multi-source join Mostly Second read-data chain-break rule unstated; post-join field scoping undefined
   c. API service Partially Guard inline policy syntax ambiguous; call.response → write-data.rows type bridge missing; validate.schema type undefined
   d. Streaming/windowing Partially All stream source config shapes missing; window.size format missing; windowed-to-group semantics vague
   e. Context inheritance Yes Minor: per-intent-field merge not illustrated
   f. Error handling Mostly @ops.circuit_breaker/fallback shapes missing; @ops at rod level vs panel level unclear
   Shorthand / chaining rules
   The implicit chaining rules are clear for linear pipelines. The from: [a, b] multi-input form is correctly demonstrated. However Rule 2 has a silent exception: it says "each subsequent rod reads the previous rod's default output" — but read-data rods have in: —, so a second read-data in a panel must be silently exempt. This is shown by example but never stated as a rule.

Decorators
@dp, @access, and @ops are clear enough to use. @sec has only its field names listed (encryption?, classification?, audit?) without types or values — it cannot be written correctly from this document alone. @cert is similarly underspecified (hash, version formats unknown). @ops sub-fields beyond retry and timeout are names-only.

Credentials and references
env(), secret_ref, policy(), adc {}, and @name/@name { override } are all well-specified and unambiguous.

3. Clarity and Completeness Audit
   a. Ambiguities
   Ambiguity Reading A Reading B Likely intent
   Guard policy syntax policy: opa: data.authz.allow (arg value is an expression string with opa: prefix) policy: policy("opa-policy-name") (named policy reference via hub) Reading A — arg.policy is an inline expression matching the expression shorthand table
   Implicit chain for second read-data Rule 2 applies → compiler error (read-data has no input) read-data always behaves as a source, breaking chain unconditionally Reading B — supported by the join example but unstated
   @access intent field merging Panel @access { purpose, operation } inherits basis from context field-by-field Panel @access block fully replaces context @access.intent Reading A — consistent with "field-level merge, panel wins on conflict" for @dp/@ops, but the access section doesn't repeat this
   from: dot notation f.reject resolves to f.err.reject (err channel) f.reject resolves to f.out.reject (non-default output channel) Reading B — filter's reject is in out, not err per the rod table
   b. Missing Grammar
   The following constructs appear in examples or tables but have no syntax rule or config shape:

DataTarget — write-data uses cfg.target: DataTarget but the document only defines the DataSource union tree. No DataTarget tree is shown. It can be inferred from DataSource but is not stated.

Stream source configs — stream.kafka, stream.pubsub, stream.kinesis are named in the DataSource union tree but no config field list is provided (contrast with postgres: host, port, db_name, tls, credentials).

@ops sub-field types — circuit_breaker, rate_limit, fallback listed as optional but no value type defined.

@sec field types — encryption?, classification?, audit? listed but no values or types defined.

validate.schema — used as cfg.schema but the type (inline schema block? registry URI? named reference?) is never defined.

pseudonymize.algo — only "sha256" appears in examples; valid values are not enumerated.

encrypt.key_ref — field mentioned, format completely undefined.

window.size / window.slide — format of duration values never shown.

StateBackend — used in store.cfg.backend but the type is never defined.

guard.out.modified — appears in the knot table (out.allowed + out.modified) with no explanation of when/what data flows to modified vs allowed.

COLLECT — listed as a reserved word but never used or defined anywhere.

c. Contradictions
Item Description
Verbose separator The verbose full example uses cfg.mode = "scan" (equals sign), while the shorthand equivalence rule shows field: val (colon). The Grammar Essentials never defines the = assignment form used in verbose rods. The normalization note says both normalize to the same AST, but the grammar key rules only show the shorthand colon form.
@access shorthand rule Shorthand Rule 4 says "intent fields can be flattened (drop the intent: wrapper)." The context strux.context syntax example still uses the full intent: { purpose, basis, operation } form inside @context, but the panel shorthand example uses the flat form. This is self-consistent but easily misread — a note clarifying that @context blocks use verbose form while @panel blocks use flat form would prevent ambiguity.
d. Dangling References
Item Status Recommendation
@when Reserved, "defined in deeper spec modules" Keep reservation, the scope note is adequate
@adapter Same Same
COLLECT Reserved word, never used or explained Add a one-line deferred note or remove from this document
guard.out.modified Appears in knot table, no explanation Define: what transformation or annotation produces modified vs allowed?
StateBackend Type name in store rod, undefined Add a one-liner (e.g., StateBackend = redis | memcached | dynamodb) or a scope note
policy("name") resolution References design-notes.md §5 for resolution mechanism Acceptable deferral, but the hub/OPA/Cedar distinction warrants a one-line inline note
e. Edge Cases
Edge case Covered? Notes
retry + fallback in @ops — which runs first? ✅ Yes "apply @ops { retry } first … then @ops { fallback } if retries exhausted." Clear.
write-data failure vs reject — difference and wiring? ✅ Yes "failure = transport/system error; reject = data-level rejection, wire like non-default output." Clear.
join in implicit chain position — second input source? ⚠️ Partial Only the explicit from: [a,b] form is shown. What happens if from: is omitted on a join rod is never stated (compile error? single-input degenerate join?).
filter → two downstream rods consuming match and reject ⚠️ Partial Rule 5 states non-default outputs need explicit from:. The case where two rods both want match (fan-out on default output) is never addressed. Can two rods share the same implicit predecessor?
Panel with no @access and no inherited context ✅ Yes "Empty AccessContext → deny (fail-closed)." Clear.
f. Token Efficiency
The document is notably lean for its coverage. A few areas where tokens could be recovered:

Verbose full-panel example (~336 tokens): the document itself calls shorthand "recommended for authoring." The verbose example is valuable for spec completeness, but its 336-token form could be cut to a 50-token annotated skeleton pointing to the shorthand version, saving ~280 tokens with no information loss.

Redundant shorthand statement: "Verbose and shorthand produce identical AST" appears both in Panel Structure (implicitly, via the equivalence code block) and in Grammar Essentials (Normalization section). One instance suffices.

Operator precedence table: for an LLM system prompt, this 8-row table could be a one-liner: Precedence: () > NOT/-unary > \*/ % > +- > comparisons > IN/BETWEEN/LIKE/IS/HAS > AND > OR. Saves ~10 rows of markdown.

DataSource note repeated: the type path concept is explained in "Type Paths" and then re-illustrated via the DataSource union tree. The inline config pattern example is valuable, but the prose explanation is redundant.

Two "Ref:" lines for panel-shorthand.md: once after Panel Structure, once after the Default Knots table. Consolidate to one.

4. Summary
   Generation Readiness: Mostly
   An LLM can generate valid .strux source for standard linear ETL pipelines, context-inheriting panels, and multi-source joins. The core 70% of the language is covered. Gaps block confident generation for stream-source pipelines, any use of @sec, @cert, or store, and any pipeline requiring validate or encrypt.

Coverage Score
Use case Verdict
a. Basic ETL ✅ Fully covered
b. Multi-source join ✅ Mostly covered (minor inference)
c. API-triggered service ⚠️ Partially covered (guard syntax, type bridge, schema type)
d. Streaming + windowing ❌ Significantly blocked (stream config, window size format)
e. Context inheritance ✅ Fully covered
f. Error handling / resilience ⚠️ Mostly covered (@ops sub-fields missing)
4 out of 6 use cases writable with confidence; 2 require material guessing.

Top 5 Issues
DataTarget type undefined (highest impact) — write-data appears in every pipeline. The document defines DataSource with a full union tree and config examples for postgres and bigquery, but never defines DataTarget. An LLM must mirror the DataSource tree by analogy. A two-line union tree + one stream config example would close this.

Stream source config shapes missing — stream.kafka, stream.pubsub, stream.kinesis appear in the DataSource union tree and in examples, but their config fields are never listed. Cases d and f cannot be written correctly without guessing field names like brokers, topic, group_id.

Second read-data implicit chain rule unstated — Multi-source panels (any join use case) depend on the compiler silently ignoring the implicit chain rule for read-data rods beyond the first. The semantics example shows this works, but no rule covers it. A single sentence — "rods whose in knot is — (source rods) are always chain-roots regardless of declaration position" — would close the gap.

@ops sub-field types absent — circuit_breaker, fallback, and rate_limit are listed as fields but have no type, shape, or example. Every resilience pattern (use case f) requires guessing their structure.

Guard policy inline syntax ambiguous — The rod table separates cfg.policy: PolicyRef and arg.policy (shorthand), but the document never shows a complete guard { policy: <expr> } shorthand assignment. The expression table shows opa: data.authz.allow in isolation, but whether the full form is policy: opa: data.authz.allow, policy: "opa: data.authz.allow", or policy: policy("opa-ref") is unresolvable from the document alone.

Verdict
Not quite ready for a self-sufficient generation reference, but one focused editing pass would get it there. The structure is solid, the core abstractions are well-expressed, and the token efficiency is good. The document needs: (1) a DataTarget union tree mirroring DataSource, (2) config field listings for all three stream sources, (3) one sentence clarifying the chain-break rule for source rods, and (4) explicit type definitions for @ops sub-fields and validate.schema. None of these require architectural changes — they are additive and localized. After that pass, generation readiness would reach Yes for all six use cases.
