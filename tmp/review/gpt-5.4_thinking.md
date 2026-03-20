Mostly: this reference is strong enough for simple linear ETL and context inheritance, but it is not yet self-sufficient for all six target use cases because several essential constructs are underspecified, missing, or only shown by example. Using only this document, I can write plausible .strux for each case, but only a small subset feels fully grounded rather than inferred.
​

Generation test
These are best-effort .strux attempts written from the reference alone. After each one, I separate what the document explicitly supports from what I had to infer or invent.
​

a. Basic ETL pipeline
text
@panel basic-etl {
@dp {
controller: "Acme",
controller_id: "ES-B12345678",
dpo: "dpo@acme.com",
record: "RPA-ETL-001"
}

@access {
intent: {
purpose: "analytics_export",
basis: "legitimate_interest",
operation: "read"
},
scope: policy("analytics-export")
}

users = read-data {
source: db.sql.postgres {
host: env("PG_HOST"),
port: 5432,
db_name: "app",
tls: true,
credentials: secret_ref { provider: vault, path: "kv/data/pg/app" }
},
mode: "scan",
fields: [id, full_name, email, phone, country]
}

by_country = filter {
predicate: country == "ES"
}

masked = pseudonymize {
algo: "sha256",
fields: ["full_name", "email", "phone"]
}

sink = write-data {
target: db.sql.bigquery {
project: "analytics-prod",
dataset: "users_curated",
location: "EU",
credentials: adc {}
}
}

dlq = write-data {
target: stream.pubsub {
project: "analytics-prod",
topic: "bq-reject-dlq"
},
from: sink.reject
}
}
Confident: read-data, filter, pseudonymize, write-data, implicit linear chaining, explicit non-default from: sink.reject, env(...), secret_ref, and adc {} are all directly supported, and the full example shows nearly the same ETL shape with a DLQ.
​

Inferred: I treated fields on read-data as enough to shape a user row and assumed the BigQuery target can be used with only project, dataset, location, and credentials because that is how the example is written.
​

Guessed: the reference never defines how to specify the Postgres table being read, so “users from Postgres” is not actually expressible with a documented table or query field.
​

b. Multi-source join
text
@panel orders-by-country {
@dp {
controller: "Acme",
controller_id: "ES-B12345678",
dpo: "dpo@acme.com",
record: "RPA-JOIN-001"
}

@access {
intent: {
purpose: "sales_analytics",
basis: "legitimate_interest",
operation: "read"
},
scope: policy("sales-analytics")
}

users = read-data {
source: @production,
mode: "scan",
fields: [id, country]
}

orders = read-data {
source: @production,
mode: "scan",
fields: [user_id, amount]
}

joined = join {
mode: "inner",
on: left.id == right.user_id,
from: [users, orders]
}

shaped = transform {
mode: "project",
fields: [left.country AS country, right.amount AS amount]
}

by_country = group {
key: country
}

totals = aggregate {
fn: [SUM(amount) AS total_amount, COUNT(*) AS order_count]
}

sink = write-data {
target: @analytics { dataset: "order_totals_by_country" }
}
}
Confident: the document explicitly defines join, group, aggregate, transform, multi-input from: [users, orders], join predicates with left and right, and aggregate expressions such as SUM(...) and COUNT(\*).
​

Inferred: I projected left.country and right.amount after the join because the document defines projection syntax but never states the exact field shape of join.out.joined.
​

Guessed: the reference does not define how users and orders are selected as two different Postgres tables, so the central “read two different tables” part is underspecified.
​

c. API-triggered service
text
@type ApiPayload {
user_id: string,
action: string,
token: string
}

@panel api-triggered-service {
@dp {
controller: "Acme",
controller_id: "ES-B12345678",
dpo: "dpo@acme.com",
record: "RPA-API-001"
}

@access {
intent: {
purpose: "request_processing",
basis: "legitimate_interest",
operation: "write"
},
scope: policy("service-write")
}

inbound = receive {
trigger: http { method: "post", path: "/events" },
timeout: "15s"
}

valid = validate {
schema: ApiPayload
}

authz = guard {
policy: policy("opa-authz-allow")
}

outbound = call {
target: http {
base_url: "https://partner.example.com",
tls: true
},
method: "post",
path: "/sync"
}

persist = write-data {
target: db.sql.postgres {
host: env("PG_HOST"),
port: 5432,
db_name: "service",
tls: true,
credentials: secret_ref { provider: vault, path: "kv/data/pg/service" }
}
}
}
Confident: receive, validate, guard, call, HTTP triggers, HTTP service targets, named policy references, and linear chaining are all present in the reference.
​

Inferred: I used policy("opa-authz-allow") to represent an OPA-backed policy because the document says policy("name") can resolve through a policy store and also shows opa: policy expressions, but it does not clearly connect the two in rod syntax.
​

Guessed: the exact syntax for validate.schema, the storage destination for write-data into Postgres, and any request/response body mapping rules are not defined here.
​

d. Streaming pipeline with windowing
text
@panel regional-stream-counts {
@dp {
controller: "Acme",
controller_id: "ES-B12345678",
dpo: "dpo@acme.com",
record: "RPA-STREAM-001"
}

@access {
intent: {
purpose: "operational_analytics",
basis: "legitimate_interest",
operation: "read"
},
scope: policy("stream-analytics")
}

events = read-data {
source: stream.kafka {
brokers: env("KAFKA_BROKERS"),
topic: "user-events",
credentials: secret_ref { provider: vault, path: "kv/data/kafka/client" }
},
mode: "stream"
}

win = window {
kind: "sliding",
size: "5m"
}

by_region = group {
key: region, window_start
}

counts = aggregate {
fn: COUNT(\*)
}

routes = split {
routes: {
eu: region IN ("ES", "FR", "DE"),
us: region == "US",
other: \*
}
}

eu_sink = write-data {
target: stream.kafka { topic: "counts-eu" },
from: routes.eu
}

us_sink = write-data {
target: stream.kafka { topic: "counts-us" },
from: routes.us
}

other_sink = write-data {
target: stream.kafka { topic: "counts-other" },
from: routes.other
}
}
Confident: read-data in stream mode, window, group, aggregate, split, and explicit routing from named split outputs are all supported at least at the rod level.
​

Inferred: I assumed windowed records expose fields such as window_start, because the reference defines window.out.windowed but not the structure of the emitted element.
​

Guessed: Kafka config fields, Kafka as a write-data target, and the hop/slide parameter for a true sliding window are not defined in this document, even though window.kind: sliding exists.
​

e. Context-heavy project
strux.context

text
@context {
@dp {
controller: "Acme",
controller_id: "ES-B12345678",
dpo: "dpo@acme.com"
}

@access {
intent: { basis: "legitimate_interest" },
scope: policy("default-read")
}

@source production = db.sql.postgres {
host: env("PG_HOST"),
port: 5432,
db_name: "app",
tls: true,
credentials: secret_ref { provider: vault, path: "kv/data/pg/app" }
}

@target analytics = db.sql.bigquery {
project: "analytics-prod",
dataset: "default_dataset",
location: "EU",
credentials: adc {}
}
}
users-eu.strux

text
@panel users-eu {
@dp { record: "RPA-CTX-001" }
@access { purpose: "geo_segmentation", operation: "read" }

users = read-data { source: @production, mode: "scan" }
eu = filter { predicate: country IN ("ES", "FR", "DE") }
sink = write-data { target: @analytics { dataset: "users_eu" } }
}
users-us.strux

text
@panel users-us {
@dp { record: "RPA-CTX-002" }
@access { purpose: "geo_segmentation", operation: "read" }

users = read-data { source: @production, mode: "scan" }
us = filter { predicate: country == "US" }
sink = write-data { target: @analytics { dataset: "users_us" } }
}
Confident: this is the best-covered use case because the document explicitly defines strux.context, inheritance rules, named @source and @target, inline override syntax like @analytics { ... }, and flattened @access in panels.
​

Inferred: I assumed panels can safely omit inherited controller, controller_id, dpo, basis, and scope, because the merge rules say those blocks inherit and panel values override only on conflict.
​

Guessed: again, the actual table or write destination inside Postgres and BigQuery is not formally described beyond broad config examples.
​

f. Error handling and resilience
text
@panel resilient-pipeline {
@dp {
controller: "Acme",
controller_id: "ES-B12345678",
dpo: "dpo@acme.com",
record: "RPA-OPS-001"
}

@access {
intent: {
purpose: "analytics_export",
basis: "legitimate_interest",
operation: "read"
},
scope: policy("analytics-export")
}

@ops { timeout: "30s" }

@ops { retry: 3, fallback: @backup_pg }
source = read-data {
source: @production,
mode: "scan"
}

clean = filter {
predicate: email IS NOT NULL
}

filter_dlq = write-data {
target: stream.pubsub {
project: "analytics-prod",
topic: "filter-rejects"
},
from: clean.reject
}

sink = write-data {
target: @analytics { dataset: "clean_users" }
}

write_rejects = write-data {
target: stream.pubsub {
project: "analytics-prod",
topic: "write-rejects"
},
from: sink.reject
}

write_failures = write-data {
target: stream.pubsub {
project: "analytics-prod",
topic: "write-failures"
},
from: sink.failure
}
}
Confident: the document explicitly states error propagation order as wired err, then retry, then fallback, and it clearly distinguishes write-data.failure from write-data.reject.
​

Inferred: routing clean.reject, sink.reject, and sink.failure through explicit from: follows the general non-default-output rule, even though examples only show reject from filter and not failure from write-data.
​

Guessed: rod-level placement of @ops and the actual type of fallback are not defined anywhere in this reference, so this is the weakest “validity” example of the six.
​

Coverage test
The document does not cover 90 percent of these use cases as a self-sufficient generation reference. It covers the shape of graphs well, but it does not cover several common adapter and rod-configuration details well enough to avoid guessing.
​

a. Partially covered: linear chaining, filter, pseudonymize, write-data, DLQ wiring, and credentials are clear, but the reference never defines how to name the Postgres table being read.
​

b. Partially covered: join, from: [a, b], grouping, and aggregation are clear, but reading two specific Postgres tables is not, and the post-join row shape is only implied.
​

c. Partially covered: receive, validate, guard, and call exist, but schema references, OPA usage in a rod field, and database write mapping are underspecified.
​

d. Mostly not covered: window and split exist, but the document does not define Kafka config shape, sliding-hop semantics, or the schema of windowed outputs.
​

e. Mostly covered: strux.context, inheritance, named references, override syntax, and flattened @access are clear enough to use with little ambiguity.
​

f. Mostly not covered: the error model is documented well, but rod-level decorator syntax and fallback configuration shape are missing, which blocks confident generation.
​

On the specific checklist: shorthand is clear for simple linear chains, default knots and explicit from: are clear for one default path plus named non-default branches, and credential/reference forms such as env, secret_ref, policy, adc, and @name are mostly clear in isolation. The weak areas are decorators, multi-input behavior beyond the single join example, and concrete adapter schemas for sources, targets, triggers, and persistence destinations.
​

Audit
The main problems are not with the idea of the language but with missing low-level generation rules. An LLM can follow the examples, but the moment a use case leaves the exact example path, ambiguity appears quickly.
​

a. Ambiguities
Enum or string values: ReadMode, CallMethod, and join.mode are presented as bare symbolic values such as scan and inner, but examples use quoted strings such as "scan" and "inner". The intended reading seems to be “quoted strings are accepted,” because every concrete example uses quotes.
​

Knot references: the document shows snap db.out.rows -> in.data, shorthand from: db.rows, and later from: f.reject, so there are two plausible readings, either shorthand drops .out or any rod.knot is accepted regardless of namespace. The intended reading appears to be the shorter rod.knot form for shorthand.
​

@access fields: the formal decorator shape shows intent: { purpose, basis, operation }, while shorthand examples flatten only purpose and operation, leaving it unclear whether basis is optional or merely inherited. The intended reading seems to be “basis may be inherited, but if nothing is inherited the empty context denies access.”
​

Multi-output defaults: split lists named route outputs and the semantics say implicit chaining follows only the default output, but the default output for split is not concretely defined. The intended reading is probably that split branches should always be wired explicitly.
​

b. Missing grammar
DataTarget is used by write-data, but only the DataSource union tree is defined in this reference.
​

There is no documented syntax for selecting a database table, view, or SQL text in read-data, even though that is needed for normal Postgres usage.
​

There is no documented syntax for selecting the destination table or insert/update mode in write-data to a database target.
​

Rod-level decorator attachment is missing, even though @ops, @sec, and @cert clearly exist as reusable component-level constructs.
​

validate.schema, guard.cfg.policy versus arg.policy, StateBackend, PolicyRef, HTTP auth/header shapes, and fallback value shape are all referenced but not fully defined.
​

c. Contradictions
The strongest contradiction is that write-data lists both failure and reject in the error column, while the semantics later say reject is “a typed branch, not a failure channel.”
​

Another contradiction is the enum presentation: symbolic values are defined without quotes, but all concrete examples use quoted strings for the same fields.
​

There is also a softer contradiction between “self-sufficient entry point” and repeated instructions to load deeper specs whenever compact rules are insufficient, because several common cases do in fact require those deeper specs.
​

d. Dangling references
@when and @adapter are reserved here but explicitly deferred to deeper specs, so they should either be removed from this compact reference or marked as out of scope more prominently.
​

DataTarget, PolicyRef, StateBackend, GcpCredentials, snap.lock, and the “hub or policy store” are all mentioned without enough local definition for self-sufficient generation.
​

I would keep them only if each gets a one-line local definition; otherwise they should be deferred with a scope note.
​

e. Edge cases
A rod with both retry and fallback is the one edge case the document answers clearly: retries run first, then fallback if retries are exhausted.
​

write-data.failure versus write-data.reject is also clear in principle: failure is transport or system error, while reject is a typed data-level branch that should be wired explicitly like any other non-default output.
​

A join in implicit chain position is not actually resolved by the document, because multi-input rods are only given an explicit from: [rod1, rod2] rule.
​

A filter followed by two downstream rods resolves only partly: implicit chaining follows match, while reject must be wired explicitly, so a mixed branch needs at least one explicit from:.
​

A panel with no @access and no inherited access context is answered clearly: empty access context denies by default.
​

f. Token efficiency
The document repeats shorthand equivalence in both “Panel Structure” and “Normalization,” and it repeats pushdown material in both “Pushdown Rules” and “Pushdown & Fusion.”
​

It also duplicates the same ETL concept in a long verbose example and then again in shorthand, which is useful once but expensive in a prompt-sized reference.
​

The spec map, repeated “load deeper spec” notes, horizontal-rule structure, and long reserved-word list add tokens, while several missing generator-critical rules would be more valuable than this overhead.
​

Summary
Generation readiness: mostly, but not reliably enough for common production generation without extra guessing, especially around database/table targeting, adapter schemas, rod-level decorators, and service or window details. Strictly interpreted, only the context-inheritance use case is fully covered without guessing, which is about 17 percent of the six cases; if you count the provided ETL example as covering a table-less scan workflow, the practical ceiling rises to about 33 percent.
​

Coverage: the graph language itself is well explained, but the “common 90%” operational details are not. The missing pieces are mostly not high-level semantics but generator-critical syntax for targets, tables, schemas, fallback values, and rod-level decorator placement.
​

Top 5 issues, from most to least impactful on generation correctness:

No syntax for selecting database tables, queries, or write destinations in ordinary DB workflows.
​

No grammar for rod-level decorators, especially @ops, plus no defined fallback shape.
​

DataTarget and several adapter or policy-related types are referenced but not defined locally.
​

Inconsistent notation for enums and knot references, which invites incompatible generations.
​

Windowing and API details are too shallow for common streaming and service use cases.
​

Verdict: this document needs another editing pass before it can serve as a truly self-sufficient LLM generation reference. It is a strong compact overview of the model, but not yet a dependable standalone source for correct .strux generation across the six representative use cases you provided.
