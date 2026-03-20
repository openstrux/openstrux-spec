1. Generation Test — Can You Use This?
   I attempted to write valid .strux source for the six use cases using only the given syntax reference. Below is the code I produced, followed by a self-assessment of Confident (explicit rule/example), Inferred (implied but filled gap), or Guessed (no coverage, made up).

a. Basic ETL pipeline
strux
@panel etl-pipeline {
@dp { record: "ETL-001" }
@access { purpose: "analytics", operation: "write" }
users = read-data {
source: db.sql.postgres {
host: env("DB_HOST"),
port: 5432,
db_name: "production",
tls: true,
credentials: secret_ref { provider: env, path: "PG_CREDENTIALS" }
},
mode: "scan"
}
filtered = filter {
predicate: address.country IN ("ES", "FR", "DE") AND deleted_at IS NULL
}
masked = pseudonymize {
algo: "sha256",
fields: ["full_name", "email", "national_id"]
}
sink = write-data {
target: db.sql.bigquery {
project: "my-project",
dataset: "analytics",
location: "EU",
credentials: adc {}
}
}
dlq = write-data {
target: stream.pubsub {
project: "my-project",
topic: "dlq"
},
from: filtered.reject
}
}
Self-assessment

Confident: @panel, @dp, @access, read-data/write-data with source/target, filter/pseudonymize with predicate/fields, from: filtered.reject. All are directly shown in the full example.

Inferred: Using inline source: and target: with field blocks (e.g., db.sql.postgres { host: ... }). The shorthand rules say “cfg./arg. prefixes are optional”, so source: is inferred.

Guessed: None for this case.

b. Multi-source join
strux
@panel join-aggregate {
@dp { record: "JOIN-001" }
@access { purpose: "analytics", operation: "read" }
users = read-data {
source: db.sql.postgres { host: env("DB_HOST"), port: 5432, db_name: "production", tls: true, credentials: secret_ref { provider: env, path: "PG_CREDENTIALS" } },
mode: "scan"
}
orders = read-data {
source: db.sql.postgres { host: env("DB_HOST"), port: 5432, db_name: "production", tls: true, credentials: secret_ref { provider: env, path: "PG_CREDENTIALS" } },
mode: "scan"
}
joined = join {
mode: "inner",
on: left.user_id == right.id,
from: [users, orders]
}
grouped = group {
key: country
}
aggregated = aggregate {
fn: [SUM(amount) AS total, COUNT(*) AS order_count]
}
sink = write-data {
target: db.sql.bigquery { project: "my-project", dataset: "analytics", location: "EU", credentials: adc {} }
}
}
Self-assessment

Confident: join with mode, on, from: [users, orders] (explicit multi-input example given). group with key, aggregate with fn using list and SQL‑like expressions.

Inferred: Implicit chain from joined to grouped – the document says “each subsequent rod without from: reads the previous rod’s default output”. Here grouped has no from:, so it reads joined’s default out (joined).

Guessed: None.

c. API-triggered service
strux
@panel api-service {
@dp { record: "API-001" }
@access { purpose: "api", operation: "write" }
req = receive {
trigger: http { method: "POST", path: "/users" }
}
validated = validate {
schema: @type User // GUESS: schema format not defined
}
authorized = guard {
policy: opa: data.authz.allow
}
api_call = call {
target: http { base_url: "https://api.example.com" },
method: "post"
}
db_write = write-data {
target: db.sql.postgres { host: env("DB_HOST"), port: 5432, db_name: "production", tls: true, credentials: secret_ref { provider: env, path: "PG_CREDENTIALS" } }
}
}
Self-assessment

Confident: receive with trigger: http, guard with policy: opa:..., call with target: http, method, implicit chaining through default knots.

Inferred: Implicit chain from receive (default out request) to validate, etc. The default knot table lists request for receive, valid for validate, allowed for guard, response for call – all used.

Guessed: The validate rod’s schema field. The document only mentions cfg.schema but gives no syntax for schemas (type reference, inline definition, etc.). I used @type User as a guess, but it is not defined.

d. Streaming pipeline with windowing
strux
@panel streaming-window {
@dp { record: "STREAM-001" }
@access { purpose: "analytics", operation: "write" }
source = read-data {
source: stream.kafka {
bootstrap_servers: "kafka:9092",
topic: "events",
consumer_group: "my-group"
},
mode: "stream"
}
windowed = window {
kind: "sliding",
size: "10s" // GUESS: unit/syntax not defined
}
counts = aggregate {
fn: COUNT(_)
}
splitter = split {
routes: {
eu: region == "EU",
us: region == "US",
other: _
}
}
sink_eu = write-data {
target: stream.pubsub { project: "my-project", topic: "eu-events" },
from: splitter.eu
}
sink_us = write-data {
target: stream.pubsub { project: "my-project", topic: "us-events" },
from: splitter.us
}
sink_other = write-data {
target: stream.pubsub { project: "my-project", topic: "other-events" },
from: splitter.other
}
}
Self-assessment

Confident: read-data with stream.kafka (union mentioned), split with routes (example given), aggregate, and wiring from: splitter.eu (named output knot).

Inferred: window rod with kind and size – the document lists cfg.kind and cfg.size but gives no examples or valid values for size.

Guessed: Kafka config fields (bootstrap_servers, topic, consumer_group) – not defined in the document. The size value "10s" is a guess; no unit specification.

e. Context-heavy project
strux.context

strux
@context {
@dp { controller: "Acme Inc", controller_id: "US-12345", dpo: "privacy@acme.com" }
@access { intent: { basis: "legitimate_interest" }, scope: policy("default-read") }
@source production = db.sql.postgres {
host: env("DB_HOST"),
port: 5432,
db_name: "production",
tls: true,
credentials: secret_ref { provider: vault, path: "db/prod" }
}
@target analytics = db.sql.bigquery {
project: "acme-analytics",
dataset: "reports",
location: "US",
credentials: adc {}
}
@ops { retry: 3, timeout: "30s" }
}
Panel 1 (user-report.strux)

strux
@panel user-report {
@dp { record: "UR-001" }
@access { purpose: "user_analytics", operation: "read" }
users = read-data { source: @production, mode: "scan" }
filtered = filter { predicate: active == true }
sink = write-data { target: @analytics { dataset: "active_users" } }
}
Panel 2 (order-report.strux)

strux
@panel order-report {
@dp { record: "OR-001" }
@access { purpose: "order_analytics", operation: "read" }
orders = read-data { source: @production, mode: "scan" }
aggregated = aggregate { fn: COUNT(\*) }
sink = write-data { target: @analytics { dataset: "order_counts" } }
}
Self-assessment

Confident: The @context block syntax is exactly as shown in the document. Panel inheritance uses @source production, @target analytics, and overrides @dp/@access as shown in the full shorthand example.

Inferred: Overriding a target field (@analytics { dataset: ... }) is shown in the shorthand example.

Guessed: None.

f. Error handling and resilience
strux
@panel error-handling {
@dp { record: "ERR-001" }
@access { purpose: "analytics", operation: "write" }
@ops { retry: 3, fallback: fallback_source } // GUESS: per-rod @ops placement, fallback reference syntax
source = read-data {
source: db.sql.postgres { host: env("DB_HOST"), port: 5432, db_name: "production", tls: true, credentials: secret_ref { provider: env, path: "PG_CREDENTIALS" } },
mode: "scan"
}
fallback_source = read-data {
source: db.sql.postgres { host: env("DB_FALLBACK_HOST"), port: 5432, db_name: "fallback", tls: true, credentials: secret_ref { provider: env, path: "PG_FALLBACK_CREDENTIALS" } },
mode: "scan"
}
filtered = filter {
predicate: status != "invalid"
}
dlq = write-data {
target: stream.pubsub { project: "my-project", topic: "filter-rejects" },
from: filtered.reject
}
sink = write-data {
target: db.sql.bigquery { project: "my-project", dataset: "analytics", location: "EU", credentials: adc {} }
}
error_topic = write-data {
target: stream.pubsub { project: "my-project", topic: "write-rejects" },
from: sink.reject
}
}
Self-assessment

Confident: Wiring filtered.reject and sink.reject is shown in the full example. The write-data error knot distinction is explained.

Inferred: The @ops decorator on a rod – the document mentions @ops in context inheritance and says “Panel can override”, but it does not explicitly show a rod‑level @ops. However, the error propagation section says “If err knot is unwired → apply @ops { retry } first”, implying @ops can be attached to rods.

Guessed: The exact placement of @ops before a rod, the syntax for fallback: fallback_source, and the behavior when both @ops and an explicit err wire are present are not specified.

2. Coverage Test — Does It Cover 90% of Use Cases?
   Use Case Covered Without Guessing Missing/Underspecified
   a. Basic ETL ✅ Yes –
   b. Multi-source join ✅ Yes –
   c. API‑triggered service ❌ No Validate rod’s schema syntax; no definition for inline or type‑referenced schema.
   d. Streaming windowing ❌ No stream.kafka config fields; window rod’s size format (units, duration syntax).
   e. Context inheritance ✅ Yes –
   f. Error handling ❌ No Per‑rod @ops placement; fallback reference syntax; interaction between @ops and wired err knots.
   Overall: The document covers most common constructs (sources, targets, basic rods, decorators, inheritance, implicit chaining). However, the validate rod, streaming sources, window rod parameters, and per‑rod @ops/fallback are either missing or too vague for an LLM to generate correct code without guessing. That leaves about 70‑80% coverage for the six use cases.

Constructs that need better specification:

validate: how to express a schema (type reference, inline structure, JSON Schema, etc.).

stream.kafka/kinesis/pubsub: the required config fields (e.g., bootstrap_servers, topic, consumer_group).

window: valid values for size (duration strings like "10s", "5m", numeric with unit?).

@ops on individual rods: syntax, whether it appears as a decorator before the rod, and the exact form of fallback.

Error wiring for rods with multiple error knots (failure vs reject): examples showing both being used simultaneously.

3. Clarity and Completeness Audit
   a. Ambiguities
   Implicit chain with multi‑output rods

Ambiguity: “Implicit chain follows the default output only (match, not reject).” What about a rod like split that has no default output? The document says split has named outputs, but does not specify a default. How does implicit chain behave after a split?

Intended: Probably the next rod must use an explicit from: because there is no default.

@ops placement

Ambiguity: The document says @ops is inherited and can be overridden in a panel, but it never says whether it can be placed on individual rods. The error propagation section suggests it applies to rods, but the grammar and examples only show it at panel/context level.

Intended: Likely allowed on rods, but not specified.

fallback meaning

Ambiguity: In @ops { fallback: ... }, is the value a rod name (to execute as a fallback), a constant, or something else? The document does not define the syntax or semantics.

b. Missing Grammar
Schema expression for validate rod – no EBNF or examples.

Stream source configuration – no grammar for fields like bootstrap_servers, topic, etc.

Window size – no grammar for duration literals.

Per‑rod @ops – not present in the grammar or panel structure definition.

@source/@target definitions inside @context are shown but not formally added to the grammar (they appear as decorators, but the @context block grammar is not given).

c. Contradictions
Default knots table says write-data default in is rows (for data) but also elements for streams. The document says “write-data default in = rows/elements”. This is vague and could cause confusion about which input is used when the source is a stream vs a database. No rule clarifies that the rod type determines the input shape.

d. Dangling References
@when and @adapter – listed as reserved words but never explained. Should be removed or deferred with a scope note.

policy("name") – mentioned but not defined how policies are defined (e.g., in the hub or policy store). The spec map points to design-notes.md, but the self‑sufficient document should at least state that policy("name") is a compile‑time reference.

@cert decorator – listed but no explanation of its fields (scope, hash, version). It appears only in the decorator table; no example or inheritance rule is given.

@sec decorator – fields (encryption?, classification?, audit?) are named but not described.

e. Edge Cases – Does the Document Tell You What Happens?
Edge Case Covered? Notes
Rod with both retry and fallback – which runs first? ❌ No The error propagation section says “apply @ops { retry } first, then @ops { fallback }”, but does not say if they are both present in the same @ops.
write-data failure vs reject – how to wire each? ✅ Yes Explained: failure is a system error channel that follows cascade; reject is a typed output.
join rod in implicit chain – where does the second input come from? ✅ Yes The normalization example shows that from: [users, orders] must be explicit; implicit chain cannot supply both.
filter followed by two downstream rods – one using match, one reject – how is implicit chain resolved? ❌ No The document says “Implicit chain follows the default output only”. If the next rod expects the default output (match), it’s fine. But if you place a rod after filter that wants the reject output, you must use from:; the implicit chain will not magically pick reject. This is implied but not explicitly stated.
Panel with no @access and no inherited access context – what happens? ✅ Yes “Empty AccessContext → deny (fail‑closed).”
f. Token Efficiency
The document is relatively concise for the complexity of the language. However, a few sections could be shortened:

The 18 Basic Rods table could be compressed (e.g., collapse read-data/write-data into one row with a note that they share fields).

Default Knots table repeats information that is also in the rod descriptions.

Expression shorthand examples could be reduced by showing only a representative sample; the current list is comprehensive but adds tokens.

Grammar essentials could be moved to an appendix or trimmed – the document claims to be self‑sufficient, but the EBNF is not actually used in any example and adds ~200 lines of markdown that an LLM must process. A simpler summary might suffice.

Overall, the token overhead is acceptable given the goal of self‑sufficiency, but the redundant tables could be merged to save tokens without loss.

4. Summary
   Generation Readiness
   Mostly – An LLM can generate correct .strux for the majority of common patterns (ETL, joins, context inheritance) using this document. However, the gaps in validate, streaming sources, window rods, and per‑rod @ops mean that for about 30% of realistic use cases, the LLM would have to guess or invent syntax.

Coverage
~70‑80% of the six use cases were fully covered without guessing. Missing pieces:

Validate schema syntax.

Stream source configuration.

Window rod parameters.

Per‑rod @ops with fallback.

Error wiring semantics when both @ops and explicit error wires are present.

Top 5 Issues (Most to Least Impactful)
Validate rod schema – completely missing, yet essential for many pipelines.

Per‑rod @ops and fallback – no syntax or placement defined, critical for resilience.

Stream source and window rod parameters – fields and allowed values not given.

split rod’s default output – unclear, affecting implicit chaining.

Redundant content – tables overlap, adding token cost without clarity.

Verdict
Needs another editing pass to address the gaps listed above, especially the missing validate schema and per‑rod @ops syntax. With those additions and a few clarifications on split and window, the document would be ready to serve as a self‑sufficient LLM generation reference.
