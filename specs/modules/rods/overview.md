# OpenStrux v0.4 — Basic Rod Types (Revised)

## Why the Revision

The first draft was dataflow-only: map, filter, reduce, join — the Beam/Spark
worldview. But OpenStrux panels describe **backend systems**, not just data
pipelines. A backend system receives HTTP requests, calls external services,
evaluates authorization policies, orchestrates multi-step workflows, manages
state, and publishes events. None of these are "map/filter/reduce."

The research covers 15 dataflow systems (Beam, Spark, Flink, Kafka Streams,
Akka Streams, RxJS, Unix pipes, Camel, CUE, Terraform, Prisma, GraphQL, dbt,
Haskell Conduit/Pipes) AND backend pattern families (Enterprise Integration
Patterns, microservices patterns, Temporal/Step Functions, NestJS/Spring Boot
abstractions, CQRS/Event Sourcing, BPMN control flow).

---

## Classification: Rod vs Decorator vs Panel Construct

Before listing rods, we need to decide what IS a rod and what isn't.

**A rod** is a node in the graph. It has typed knots (cfg, arg, in, out, err).
It receives data, processes it, emits data. It's the unit of certification.

**A decorator** (@dp, @sec, @ops) is metadata on a rod or panel. It declares
intent or operational requirements. It does NOT process data.

**A panel construct** is a structural property of the graph itself — how rods
are wired, not what they do.

### What maps to what

| Pattern | OpenStrux mechanism | Why |
|---|---|---|
| Retry + backoff | `@ops:retry(3, "exponential")` | Execution concern, not data processing |
| Timeout | `@ops:timeout("30s")` | Execution concern |
| Circuit breaker | `@ops:circuit_breaker(true)` | State-based failure prevention — decorator on rod |
| Rate limiting | `@ops:rate_limit("100/min")` | Throttling — decorator on receive rod |
| Bulkhead | `@ops:bulkhead("pool-a", 10)` | Resource isolation — decorator on panel |
| Fallback | `@ops:fallback("cache")` | Alternative path — decorator on rod |
| Sequential composition | Snaps: `a.out → b.in` | That's just the graph topology |
| Parallel composition | Fan-out: `a.out → b.in` + `a.out → c.in` | Graph topology |
| Conditional branching | `split` rod | Rod (routes elements) |
| Saga / compensation | Panel with `@compensate` block | Panel construct (see below) |
| Health check | Panel with `@ops:health` | Decorator on panel |
| Service discovery | Adapter concern (cfg resolves at deploy time) | Not a rod |
| Load balancing | Adapter concern | Not a rod |

---

## The 18 Basic Rods

```
I/O — Data (2)       I/O — Service (3)      Computation (7)
────────────────      ─────────────────      ────────────────
read-data             receive                transform
write-data            respond                filter
                      call                   group
                                             aggregate
Control (2)           Compliance (3)         merge
────────────          ──────────────         join
guard                 validate               window
store                 pseudonymize
                      encrypt                Topology (1)
                                             ────────────
                                             split
```

---

## Category 1: I/O — Data (2 rods)

These handle bulk data: databases, streams, files. Unchanged from the
first draft.

### `read-data` — Bulk data source

Reads from any `DataSource` (Postgres, Kafka, S3, etc.). Evaluates
AccessContext. Delegates to adapter.

**Maps to:** Beam Read, Spark DataSource, dbt source

### `write-data` — Bulk data sink

Writes to any `DataTarget`. Symmetric to read-data.

**Maps to:** Beam Write, Spark DataSink, dbt materialization

---

## Category 2: I/O — Service (3 rods) ← NEW

These handle **request/response interactions** with the outside world.
This is the pattern that Express, FastAPI, Spring Boot, NestJS, gRPC
services, and serverless functions all implement.

### `receive` — Entry point (trigger)

The source rod for request/response panels. Defines HOW the panel is
triggered: HTTP request, gRPC call, event subscription, cron schedule,
queue message, webhook, or manual invocation.

| Knot direction | Key knots |
|---|---|
| cfg | `trigger: Trigger` |
| arg | `timeout: Optional<string>` |
| out | `request: Single<Request>`, `context: Single<RequestContext>` |
| err | `invalid: ErrorKnot` (malformed request) |

```
@type Trigger = union {
  http:       HttpTrigger
  grpc:       GrpcTrigger
  event:      EventTrigger
  schedule:   ScheduleTrigger
  queue:      QueueTrigger
  manual:     ManualTrigger
}

@type HttpTrigger {
  method:     HttpMethod
  path:       string                  // e.g., "/users/:id"
  headers:    Optional<Map<string, string>>
}

@type HttpMethod = enum { GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS }

@type GrpcTrigger {
  service:    string
  method:     string
  streaming:  GrpcStreamMode
}

@type GrpcStreamMode = enum { unary, server_stream, client_stream, bidi }

@type EventTrigger {
  source:     DataSource              // reuse the DataSource union
  topic:      string
  filter:     Optional<string>        // event filter expression
}

@type ScheduleTrigger {
  cron:       Optional<string>        // "0 9 * * MON-FRI"
  interval:   Optional<string>        // "5m", "1h"
  timezone:   Optional<string>
}

@type QueueTrigger {
  source:     DataSource
  queue:      string
  batch_size: Optional<number>
}

@type ManualTrigger {}

@type Request {
  id:         string
  payload:    bytes
  headers:    Map<string, string>
  params:     Map<string, string>     // URL params, query params
  ts:         date
}

@type RequestContext {
  principal:  Principal               // extracted from auth header/token
  trace_id:   string
  source_ip:  Optional<string>
}
```

**Why it's basic:** Every backend system starts with something receiving a
request. Without receive, you can only build data pipelines, not services.

**Maps to:** Express app.get(), FastAPI @app.get(), Spring @GetMapping,
NestJS @Controller, AWS Lambda handler, Temporal Workflow entry

### `respond` — Exit point (return to caller)

The sink rod for request/response panels. Sends a response back to the
entity that triggered the panel via `receive`. Only meaningful when the
panel's source is a `receive` rod.

| Knot direction | Key knots |
|---|---|
| cfg | — |
| arg | `status: Optional<number>`, `headers: Optional<Map<string, string>>` |
| in  | `data: Single<T>` |
| out | `sent: Single<ResponseReceipt>` |
| err | `failure: ErrorKnot` |

```
@type ResponseReceipt {
  request_id:   string
  status:       number
  latency_ms:   number
  ts:           date
}
```

**Why it's basic:** Request/response is the most common backend pattern.
Without respond, panels can sink data but can't answer callers.

**Maps to:** Express res.json(), FastAPI return, gRPC response,
Lambda return value

### `call` — Invoke external service (request/response)

The fundamental service-to-service communication primitive. Sends a request
to an external service and awaits a response. This is different from
read-data (which reads data stores) — call is for **APIs and services**.

| Knot direction | Key knots |
|---|---|
| cfg | `target: ServiceTarget`, `method: CallMethod` |
| arg | `path: Optional<string>`, `headers: Optional<Map<string, string>>`, `timeout: Optional<string>` |
| in  | `request: Single<T>` (or `Stream<T>` for streaming) |
| out | `response: Single<U>`, `metadata: Single<CallMetadata>` |
| err | `failure: ErrorKnot`, `timeout: ErrorKnot` |

```
@type ServiceTarget = union {
  http:       HttpTarget
  grpc:       GrpcTarget
  function:   FunctionTarget
}

@type HttpTarget {
  base_url:   string
  auth:       Optional<ServiceAuth>
  tls:        bool
}

@type GrpcTarget {
  host:       string
  port:       number
  proto:      string                  // reference to .proto service
  tls:        bool
}

@type FunctionTarget {
  provider:   FunctionProvider
  name:       string
  region:     Optional<string>
}

@type FunctionProvider = enum { aws_lambda, gcp_functions, azure_functions }

@type ServiceAuth = union {
  bearer:     BearerAuth
  api_key:    ApiKeyAuth
  mtls:       MtlsAuth
  oauth2:     OAuth2Auth
}

@type BearerAuth {
  token:      SecretReference
}

@type ApiKeyAuth {
  header:     string
  key:        SecretReference
}

@type MtlsAuth {
  cert:       string
  key:        SecretReference
}

@type OAuth2Auth {
  token_url:  string
  client_id:  string
  client_secret: SecretReference
  scopes:     Batch<string>
}

@type CallMethod = enum {
  get, post, put, patch, delete       // HTTP
  unary, server_stream                // gRPC
  invoke                              // function
}

@type CallMetadata {
  status:       number
  latency_ms:   number
  retries:      number
  target_path:  string
  ts:           date
}
```

**Why it's basic:** Microservices communicate via calls. Without this rod,
you can't build service-oriented architectures. Every service mesh, API
gateway, and BFF pattern is built on call.

**Maps to:** fetch/axios, gRPC client, AWS SDK invoke, Temporal Activity,
Step Functions Task state, Camel .to("http://...")

**Design: call vs read-data.** They look similar but serve different
purposes:

| | `read-data` | `call` |
|---|---|---|
| Purpose | Retrieve data from a store | Invoke a service operation |
| Interaction | Read-only (query) | Read or write (action) |
| Target | DataSource (db, stream, file) | ServiceTarget (API, function) |
| Response | Stream of records | Single response (or stream) |
| Idempotent | Always | Depends on method |
| Typical use | "Get me all users" | "Create an order" / "Send an email" |

---

## Category 3: Computation (7 rods)

The dataflow primitives. Unchanged from the first draft — these are the
irreducible set validated across 15 systems.

### `transform` — MAP / FLATMAP (15/15 systems)

Element-wise transformation. 1:1 (map) or 1:N (flat_map).

### `filter` — Predicate selection (15/15 systems)

Dual output: match + reject. Critical for data minimization.

### `group` — Key-based partitioning (13/15 systems)

Partition stream by key function.

### `aggregate` — Associative reduction (13/15 systems)

Combine grouped elements. Built-in: count, sum, avg, min, max.

### `merge` — N inputs → 1 output (13/15 systems)

Union of same-type streams.

### `join` — Combine by key (11/15 systems)

Inner, left, right, outer, cross, lookup modes.

### `window` — Temporal grouping (6/15 — all streaming systems)

Fixed, sliding, session windows. With trigger and late-data handling.

---

## Category 4: Control (2 rods) ← NEW

### `guard` — Policy evaluation (allow / deny / modify)

The explicit authorization and policy checkpoint. While AccessContext is
implicit (available to every rod), `guard` is the rod that **evaluates**
business-level policies before allowing data to flow further.

This is the rod equivalent of NestJS Guards, Spring Security filters,
Express middleware, and OPA/Cedar policy evaluation points.

| Knot direction | Key knots |
|---|---|
| cfg | `policy: PolicyRef` |
| arg | `rules: Optional<Batch<string>>` |
| in  | `data: Stream<T>` |
| out | `allowed: Stream<T>`, `modified: Stream<T>` |
| err | `denied: Stream<GuardDenial>` |

```
@type PolicyRef = union {
  inline:     InlinePolicy
  external:   ExternalPolicyRef
}

@type InlinePolicy {
  rules:      Batch<PolicyRule>
}

@type PolicyRule {
  condition:  string                  // predicate expression
  action:     GuardAction
  reason:     string
}

@type GuardAction = enum {
  allow                               // pass through unchanged
  deny                                // reject to err.denied
  modify                              // transform and pass (e.g., redact fields)
}

@type ExternalPolicyRef {
  engine:     PolicyEngine
  policy_id:  string
}

@type PolicyEngine = enum { opa, cedar, casbin, custom }

@type GuardDenial {
  element:    bytes
  rule:       string
  reason:     string
  principal:  string
  ts:         date
}
```

**Why it's basic and NOT just a filter:**

| | `filter` | `guard` |
|---|---|---|
| Evaluates | Data content (predicates on fields) | Access policy (predicates on principal + intent + data) |
| Uses | AccessContext: no | AccessContext: yes (always) |
| Can modify | No (pass or reject) | Yes (redact, mask, enrich) |
| Audit trail | Optional | Mandatory (every denial logged) |
| Policy source | Inline predicate | External policy engine (OPA, Cedar) |
| Compliance | General | Maps to authorization controls in ENS, SOC2, ISO 27001 |

**Maps to:** NestJS @UseGuards, Spring @PreAuthorize, Express auth
middleware, OPA policy, Cedar authorization, AWS IAM policy evaluation,
Apache Ranger

### `store` — State management (get / put / delete)

Read and write individual state entries. This is NOT bulk data access
(that's read-data/write-data). Store is for **operational state**: session
data, counters, flags, workflow state, cache entries.

| Knot direction | Key knots |
|---|---|
| cfg | `backend: StateBackend`, `mode: StoreMode` |
| arg | `namespace: Optional<string>`, `ttl: Optional<string>` |
| in  | `key: Single<string>`, `value: Optional<Single<T>>` (@when mode != "get") |
| out | `result: Single<T>`, `exists: Single<bool>` |
| err | `failure: ErrorKnot`, `conflict: ErrorKnot` (@when mode = "cas") |

```
@type StateBackend = union {
  redis:      RedisConfig
  dynamodb:   DynamoConfig             // reuse from DataSource hierarchy
  memory:     MemoryConfig             // in-process (for testing/dev)
  rocksdb:    RocksDbConfig            // embedded (Kafka Streams style)
}

@type RedisConfig {
  host:       string
  port:       number  [1..65535]
  db:         Optional<number>
  tls:        bool
  credentials: Optional<SecretReference>
}

@type MemoryConfig {
  max_entries:  Optional<number>
}

@type RocksDbConfig {
  path:       string
}

@type StoreMode = enum {
  get                                  // read by key
  put                                  // write (overwrite if exists)
  delete                               // remove by key
  cas                                  // compare-and-swap (optimistic lock)
  increment                            // atomic counter increment
}
```

**Why it's basic and NOT just read-data/write-data:**

| | `read-data` / `write-data` | `store` |
|---|---|---|
| Access pattern | Bulk (scan, query, stream) | Single key (get, put, delete) |
| Cardinality | Many records | One record at a time |
| Use case | Data pipeline I/O | Operational state, caching, counters |
| Transaction | Batch-oriented | Per-key atomic (CAS, increment) |
| Lifetime | Persistent (data at rest) | Often ephemeral (TTL, session) |

**Maps to:** Kafka Streams StateStore, Flink ValueState, Redis
get/set, Temporal Workflow state, Step Functions ResultPath,
NestJS CacheModule, Spring @Cacheable

---

## Category 5: Compliance (3 rods)

Unchanged from the first draft. These are computationally transforms,
but structurally distinct for certification.

### `validate` — Schema + constraint validation

GDPR Art. 5 (accuracy). Structured error output for audit trail.

### `pseudonymize` — PII pseudonymization

GDPR Art. 4(5), Art. 25. Hash/tokenize + reversal mapping.

### `encrypt` — Field/record encryption

ENS, ISO 27001, SOC2. KMS key references, not keys.

---

## Category 6: Topology (1 rod)

### `split` — Route to N outputs by predicate

Content-based routing. Inverse of merge.

---

## What Is NOT a Basic Rod (Refined)

### Decorators (@ops — operational concerns)

```
@ops {
  retry:            3
  retry_backoff:    "exponential"
  retry_jitter:     true
  timeout:          "30s"
  circuit_breaker:  true
  cb_threshold:     5                 // failures before opening
  cb_reset:         "60s"             // time before half-open
  rate_limit:       "100/min"
  bulkhead:         "pool-a"
  bulkhead_size:    10
  fallback:         "cache"           // rod name to use on failure
}
```

All resilience patterns (retry, timeout, circuit breaker, bulkhead,
rate limiting, fallback) are decorators, NOT rods. Reason: they don't
transform data — they govern HOW a rod executes.

### Panel constructs (structural, not processing)

**Saga / compensation:**

```
@panel transfer-money {
  @meta { mode: "saga" }

  @compensate {
    debit:   call { cfg.target = http { ... }, arg.path = "/refund" }
    credit:  // no compensation needed (never executed if debit failed)
  }

  @rod auth = guard { ... }
  @rod debit = call { ... }
  @rod credit = call { ... }
  snap debit.out.response -> credit.in.request
}
```

The saga pattern is panel-level orchestration, not a rod. The panel
declares compensation actions for each step. If a step fails, the
runtime executes compensations in reverse order.

**CQRS:**

```
@panel user-service {
  @rod req = receive { cfg.trigger = http { ... } }

  @rod route = split {
    arg.routes = {
      "read":  "intent.operation == 'read'"
      "write": "intent.operation == 'write'"
    }
    snap req.out.request -> in.data
  }

  // Read path
  @rod read = read-data { snap route.out.read -> in.key ... }
  @rod read_resp = respond { snap read.out.rows -> in.data }

  // Write path
  @rod write = write-data { snap route.out.write -> in.rows ... }
  @rod emit_event = call { snap write.out.receipt -> in.request ... }
  @rod write_resp = respond { snap emit_event.out.response -> in.data }
}
```

CQRS is a topology pattern — separate read/write paths. It's expressed
via split + snaps, not via a special rod.

### Patterns (composite rods)

| Pattern | Composition |
|---|---|
| `api-gateway` | receive → guard → split (route) → call → respond |
| `scatter-gather` | split → N × call → merge → respond |
| `webhook-relay` | receive → guard → transform → call (fan-out) |
| `dp-safe-ingest` | validate → pseudonymize → split (DLQ) |
| `enrich` | read-data (lookup) → join |
| `cdc-stream` | read-data (stream) → filter → transform → call |
| `scheduled-job` | receive (cron) → read-data → transform → write-data |

### Convenience rods (useful but decomposable)

| Rod | Equivalent |
|---|---|
| `dedupe` | group(by key) → aggregate(first) |
| `sort` | window(global) → aggregate(ordered_collect) |
| `cache` | store(get) → if miss: compute → store(put) |
| `notify` | call(target = webhook/email/slack) |
| `batch` | window(count) → aggregate(collect) |
| `debounce` | window(session) → aggregate(last) |

---

## Summary: 18 Basic Rods

| # | Rod | Category | New? | Maps to |
|---|-----|----------|------|---------|
| 1 | `read-data` | I/O — Data | v0.4 | Beam Read, Spark DataSource, SQL FROM |
| 2 | `write-data` | I/O — Data | v0.4 | Beam Write, Spark DataSink, SQL INSERT |
| 3 | `receive` | I/O — Service | **NEW** | Express route, Lambda handler, Temporal Workflow |
| 4 | `respond` | I/O — Service | **NEW** | Express res.json(), Lambda return |
| 5 | `call` | I/O — Service | **NEW** | fetch/axios, gRPC client, Step Functions Task |
| 6 | `transform` | Computation | v0.4 | Beam ParDo, Spark map, SQL SELECT expr |
| 7 | `filter` | Computation | v0.4 | Beam filter, Spark filter, SQL WHERE |
| 8 | `group` | Computation | v0.4 | Beam GroupByKey, SQL GROUP BY |
| 9 | `aggregate` | Computation | v0.4 | Beam Combine, SQL SUM/COUNT |
| 10 | `merge` | Computation | v0.4 | Beam Flatten, Spark union, SQL UNION ALL |
| 11 | `join` | Computation | v0.4 | Beam CoGroupByKey, SQL JOIN |
| 12 | `window` | Computation | v0.4 | Beam Window, Flink window |
| 13 | `guard` | Control | **NEW** | NestJS Guard, OPA, Cedar, Spring Security |
| 14 | `store` | Control | **NEW** | Kafka StateStore, Redis, Flink ValueState |
| 15 | `validate` | Compliance | v0.4 | JSON Schema, Great Expectations |
| 16 | `pseudonymize` | Compliance | v0.4 | (OpenStrux-native) |
| 17 | `encrypt` | Compliance | v0.4 | KMS, Vault transit |
| 18 | `split` | Topology | v0.4 | Beam Partition, Content-Based Router |

---

## Panel Archetypes

With 18 basic rods, we can express every common backend pattern:

### Data Pipeline

```
read-data → validate → transform → filter → write-data
```

Classic ETL. The v0.3 use case.

### REST API

```
receive(http) → guard → read-data → transform → respond
```

Request/response service. The Express/FastAPI use case.

### Microservice

```
receive(grpc) → guard → store(get) → call → store(put) → respond
```

Stateful service with external dependency. The NestJS/Spring use case.

### Event-Driven Worker

```
receive(event) → guard → transform → call → write-data
```

Event consumer that processes and acts. The Kafka consumer use case.

### Scheduled Job

```
receive(schedule) → read-data → aggregate → call(webhook) → write-data
```

Cron-triggered batch process. The Airflow/Step Functions use case.

### Saga (orchestrated)

```
receive(http) → guard → call(debit) → call(credit) → respond
  @compensate { debit: call(refund) }
```

Multi-step with compensation. The Temporal/Step Functions use case.

### API Gateway

```
receive(http) → guard → split(route) → call(service-A|B|C) → respond
```

Routing + auth + forwarding. The Kong/Envoy use case.

### GDPR Pipeline

```
read-data → validate → guard → pseudonymize → encrypt → write-data
```

Full compliance pipeline. The OpenStrux-native use case.

### Real-Time Dashboard

```
receive(event) → window(5m) → aggregate → store(put) → call(websocket)
```

Stream processing + state + push. The Flink/Kafka Streams use case.

---

## Completeness Argument

**Dataflow completeness:** The 7 computational rods (transform, filter,
group, aggregate, merge, join, window) are the irreducible set identified
across 15 systems and validated by Akidau et al. (VLDB 2015).

**Backend completeness:** The 5 I/O + control rods (receive, respond,
call, guard, store) cover the 5 fundamental backend interactions:

1. **receive** — accept work (HTTP, event, schedule, queue)
2. **respond** — return results to caller
3. **call** — invoke external services
4. **guard** — enforce access policy
5. **store** — manage operational state

**Compliance completeness:** The 3 compliance rods (validate, pseudonymize,
encrypt) cover the three certification-critical operations that regulators
expect to see explicitly in data lineage.

**Topology completeness:** split + merge together enable arbitrary routing.

**Everything else** is either:

- A **decorator** (@ops for resilience, @dp for data protection)
- A **panel construct** (@compensate for saga, topology for CQRS)
- A **pattern rod** (composite of basic rods)
- A **convenience rod** (decomposable into basic rods)

18 basic rods. No more, no less.
