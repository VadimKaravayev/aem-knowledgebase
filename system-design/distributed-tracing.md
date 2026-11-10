# Distributed tracing — traces, spans, OpenTelemetry

Following one request through every service it touches, and the tools that record and display it. Docs-based. Overview of all three signals: [[observability]]; siblings: [[logging]], [[metrics-prometheus]], [[slo-error-budgets-alerting]].

## Why it exists

Metrics say a service is slow; logs say what one service did. Neither answers "this request took three seconds across nine services — where?". A trace is the record of one request's whole journey, with the time spent at each step.

## Trace and span

- A **trace** is everything that happened for one request, identified by a **trace ID** shared by every participant.
- A **span** is one unit of work inside it: handling an HTTP request, calling another service, running a query. A span has:

| Field | Content |
|---|---|
| Trace ID | Which trace it belongs to |
| Span ID | Its own identifier |
| Parent span ID | The span that caused it; empty for the root |
| Name | The operation: `POST /orders`, `SELECT orders` |
| Start time and duration | When, and how long |
| Kind | Server, client, producer, consumer or internal |
| Attributes | Key–value details: HTTP method, status code, database name |
| Events | Timestamped notes inside the span, such as an exception |
| Status | OK or error |

## Parent–child relationship

Every span except the first names its parent, which turns a flat list of spans into a tree:

```
Trace 4bf92f35…
└─ POST /checkout                    gateway        0 ─────────────────── 820 ms
   ├─ POST /orders                   order-service    10 ──────────── 610 ms
   │  ├─ SELECT customer             postgres            20 ─ 45 ms
   │  ├─ POST /payments              payment-service     50 ──────── 520 ms
   │  │  └─ POST /charge             external provider     60 ───── 500 ms
   │  └─ INSERT orders               postgres                        530 ─ 560 ms
   └─ publish order.created          broker                                 620 ─ 640 ms
```

- The **root span** has no parent and covers the whole request.
- A **child** is work done on behalf of its parent; it starts after the parent starts, and for a synchronous call ends before the parent ends.
- Read as a waterfall, the tree shows where the time went (here, the external provider), what ran in sequence that could run in parallel, and repeated calls in a loop — the N+1 query is visible at a glance.
- A remote call normally produces **two spans**: a client span in the caller and a server span in the callee. The gap between them is network and queueing time.
- Asynchronous work that outlives the request (a message consumed later) is connected by a **link** rather than strict parentage, since the "parent" has finished long before.

### Context propagation

The tree only forms if each service learns the trace ID and the calling span's ID. They travel with the request — for HTTP in the W3C `traceparent` header:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             │  │                                │                └ flags (01 = sampled)
             │  │                                └ parent span ID
             │  └ trace ID
             └ version
```

- The receiving service creates its span with that parent ID, and sends its own span ID as the parent on the calls it makes.
- **Every hop must forward it.** One service, proxy or queue that drops the header cuts the trace into two unrelated pieces. Messaging needs the context copied into message headers explicitly.
- Older systems use Zipkin's **B3** headers; most libraries read and write both.
- Inside a process the context must also follow the work across threads and asynchronous callbacks, which is where hand-written instrumentation most often loses it.

### Sampling

Recording every span of every request is too expensive at volume.

- **Head sampling** decides at the first hop (say 1%), and the decision travels in the flags. Cheap, but blind — most of the interesting requests are thrown away.
- **Tail sampling** decides after the trace is complete: keep every error and every slow trace, and a small share of the rest. It needs a collector that buffers whole traces, and it is what makes tracing useful for debugging.

## OpenTelemetry

The vendor-neutral standard for producing telemetry — traces, metrics and logs — hosted by the Cloud Native Computing Foundation. It replaced the two earlier projects, OpenTracing and OpenCensus.

| Part | What it is |
|---|---|
| **API and SDK** | Libraries per language to create spans, metrics and logs |
| **Auto-instrumentation** | Agents that instrument common frameworks and clients with no code changes (for Java, a `-javaagent`) |
| **Semantic conventions** | Standard attribute names (`http.request.method`, `db.system`) so every backend understands the data |
| **OTLP** | The wire protocol for sending telemetry |
| **Collector** | A standalone pipeline: receivers → processors (batching, filtering, tail sampling) → exporters |

- **OpenTelemetry is not a backend.** It produces and ships data; storage, search and the UI are something else (below).
- The point is **instrument once, choose the backend later**: applications emit OTLP, and switching or adding a vendor is a Collector configuration change, not a code change.
- The Collector is also where sensitive attributes are removed and volume is controlled before data leaves the network.

## Backends

### Jaeger and Zipkin

Both are open-source systems that store traces and show them as searchable waterfalls.

| | Zipkin | Jaeger |
|---|---|---|
| Origin | Twitter; modelled on Google's Dapper paper | Uber; a CNCF graduated project |
| Character | Older, simpler, a single service to run | More components, more features |
| Propagation format it introduced | B3 headers | Its own header; now uses W3C |
| Storage | In-memory, Elasticsearch, Cassandra, MySQL | Elasticsearch/OpenSearch, Cassandra, others |
| Relation to OpenTelemetry | Accepts OTel data through the Collector | Accepts OTLP directly; its own client libraries are retired in favour of the OTel SDKs |

Either is a reasonable self-hosted choice; Grafana Tempo is the third common one. All of them are for traces only — metrics and logs live elsewhere, joined by the trace ID.

### Dynatrace and Datadog

Commercial, hosted platforms that cover metrics, logs, traces, real-user monitoring and synthetic checks in one product.

| | Dynatrace | Datadog |
|---|---|---|
| Collection | One agent per host that instruments processes automatically | An agent plus a large catalogue of integrations |
| Known for | Automatic topology mapping and automated root-cause analysis | Breadth, dashboards, fast setup |
| Cost model | Consumption-based, by monitored hosts and data | Per host, plus per volume of logs, spans and **custom metrics** |

- What is bought: no backend to run, correlation between signals out of the box, and alerting, anomaly detection and dashboards that work on day one.
- What it costs: a bill that grows with data volume and metric cardinality, often faster than the infrastructure it monitors; and, if the vendor's own agents are used for instrumentation, lock-in.
- Both accept OpenTelemetry data, so instrumenting with OTel keeps the option of leaving.

## Service mesh

A **service mesh** (Istio, Linkerd) places a network proxy next to every service instance — usually a sidecar container — and routes all traffic between services through those proxies.

- Because the proxies see every request, the mesh provides **uniform telemetry with no application code**: request rate, error rate and latency for every service-to-service edge, access logs, and a live dependency map.
- It also moves cross-cutting network behaviour out of the applications: mutual TLS, retries, timeouts, traffic splitting ([[reliability-patterns]]).
- **The tracing trap**: proxies create spans for each hop, but a proxy cannot know which outgoing call belongs to which incoming request — that link exists only inside the application. **The application must still copy the trace headers from the incoming request to its outgoing ones**, or the mesh produces many two-span traces instead of one complete trace.
- The mesh sees only the network: nothing about database time, in-process work or business context. It complements instrumentation; it does not replace it.
- Costs: an extra proxy hop on every call, memory and CPU per sidecar, and a substantial piece of infrastructure to operate. Worth it for many services in several languages; heavy for a handful.

## Traps

- **A broken trace is usually a propagation gap**, not a backend problem: a hop without instrumentation, a queue, a thread pool.
- **Timestamps from different hosts disagree.** Trust the parent–child structure over the absolute times when a child appears to start before its parent.
- **Putting everything in attributes**: request bodies, tokens, personal data. Span attributes are stored and searchable, and need the same care as logs.
- **Sampling 1% at the head and expecting to find a specific failed request.** If errors matter, sample at the tail.
- **Traces without the trace ID in logs.** The pivot from a slow span to that service's log lines for the same request is where most of the diagnostic value is ([[logging]]).
