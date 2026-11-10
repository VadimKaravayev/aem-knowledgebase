# Logging — levels, structured JSON, correlation IDs, ELK

What to write to a log, in what shape, how to tie one request's lines together across services, and where the lines go. Docs-based. Overview of all three signals: [[observability]]; the sibling docs are [[metrics-prometheus]], [[distributed-tracing]] and [[slo-error-budgets-alerting]].

## Log levels

| Level | Means | Example | In production |
|---|---|---|---|
| **ERROR** | An operation failed and a person should look at it eventually | Payment provider call failed after retries; order not saved | On; feeds alerts |
| **WARN** | Something unexpected was handled, or is about to become a problem | Retry succeeded on the third attempt; fallback served; pool 90% full | On |
| **INFO** | A business or lifecycle milestone, normal operation | Service started; order created; job finished | On — the default level |
| **DEBUG** | Detail for diagnosing: inputs, branches taken, intermediate values | Cache miss for key; SQL parameters | Off; switched on per logger, temporarily |

Many frameworks add **TRACE** below DEBUG and **FATAL** above ERROR (the process cannot continue).

Rules that keep levels meaningful:

- **ERROR is a promise that someone should care.** A client sending invalid input is expected behaviour — that is a `400` and at most INFO. If ERROR includes routine events, alerts on the error rate are useless.
- **Log an exception once, where it is handled.** Catch, log and rethrow at every layer prints the same stack trace five times.
- **WARN must be actionable in aggregate**: one is noise, a rising rate is a signal. If nobody would ever act on it, it is DEBUG.
- **INFO should let someone reconstruct what the system did** without reading code, at a volume that is affordable all year.
- **Change levels at runtime**, per logger, without a restart — DEBUG for one package for ten minutes is the normal diagnostic move.

## A good log line

JSON is chosen deliberately:

```json
{
  "timestamp": "2025-12-22T14:30:00Z",
  "level": "INFO",
  "service": "order-service",
  "user_id": 123,
  "order_id": 456,
  "message": "Order created successfully"
}
```

What makes it good:

- **One event, one object.** A stack trace or multi-line value stays inside the event instead of becoming forty unrelated lines.
- **Identifiers are fields, not text.** `order_id:456` is an exact, indexed query. With `"Order 456 created for user 123"` the same question is a regular expression that breaks when someone rewords the message.
- **The message is constant.** Identical events group and count; variables live in fields.
- **The timestamp is UTC in ISO 8601**, so lines from servers in different time zones sort correctly.
- **`service` says who wrote it**, which matters as soon as logs from many services share one store.

What a production version usually adds: a **correlation or trace ID** (next section), milliseconds in the timestamp, host or instance, application version, environment, and — on a request's closing line — status and duration.

Why JSON rather than plain text:

- Parsing is exact. Plain text needs a hand-written pattern per format, and a changed log statement silently stops matching.
- Every field is searchable and aggregatable without preparation.
- New fields can be added without breaking any consumer.
- The encoder **escapes the values**: a user-supplied string containing a newline or a quote cannot break out of its field and forge a second log entry. Plain-text logs that interpolate raw input are open to exactly that.

The costs: lines are larger, and raw JSON is tiring to read at a terminal — use a readable pattern locally and JSON everywhere logs are shipped.

### Traps

- **Secrets and personal data**: passwords, tokens, session cookies, card numbers, full request bodies. Logs are read by more people and kept longer than the database. Log an internal ID, never the credential or the email.
- **Building JSON by string concatenation.** One value with a quote in it produces an unparseable line, which the pipeline then drops without an error. Use the logging library's JSON encoder.
- **Field-type conflicts**: `user_id` as a number in one service and a string in another makes the search index reject one of them. Agree on names and types across services.
- **Eager message building**: `log.debug("state: " + expensive())` pays the cost even when DEBUG is off. Use parameterized calls (`log.debug("state: {}", value)`).
- **Unbounded volume**: logging inside a hot loop, or one line per item in a batch. Log volume is a bill and an outage risk — the pipeline saturates during an incident, exactly when it is needed.
- **Writing to local files in containers.** Write to standard output and let the platform collect it; a file inside a container disappears with it.
- **Blocking appenders**: a synchronous write to a slow disk or network sink makes the logger the bottleneck. Use an asynchronous appender with a bounded buffer.

## Correlation ID

One request usually produces log lines in several services and several threads. A **correlation ID** is a unique value attached to the request at the edge and written on every line it causes, so one query returns the whole story.

- **Generate at the first hop** (gateway, load balancer, first service) if the request does not carry one; otherwise reuse it.
- **Propagate on every outgoing call**: an HTTP header (`X-Correlation-ID` and `X-Request-ID` are the common names) and a message header on anything put on a queue.
- **Attach to every log line automatically**, through the logging framework's context (MDC in SLF4J/Logback/Log4j), not by passing it to each log call.
- **Return it to the caller** in a response header and show it on error pages, so a support request arrives with the ID that finds the logs.

```java
public class CorrelationIdFilter implements Filter {
    private static final String HEADER = "X-Correlation-ID";
    private static final Pattern SAFE = Pattern.compile("[A-Za-z0-9-]{1,64}");

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        String id = ((HttpServletRequest) req).getHeader(HEADER);
        if (id == null || !SAFE.matcher(id).matches()) {
            id = UUID.randomUUID().toString();      // absent or untrusted: make our own
        }
        MDC.put("correlation_id", id);
        ((HttpServletResponse) res).setHeader(HEADER, id);
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.remove("correlation_id");           // the thread goes back to the pool
        }
    }
}
```

Traps:

- **MDC is thread-local.** Work handed to another thread (executor, `CompletableFuture`, reactive pipeline) loses the ID unless the context is copied across.
- **Not clearing it** leaks one request's ID into the next request served by the same pooled thread.
- **An incoming ID is user input.** Validate its shape and length before logging or forwarding it.
- **One hop that drops the header** splits the story in two. Asynchronous hops are where it is usually lost.
- Where distributed tracing is in place, the **trace ID is the correlation ID** — log that instead of maintaining a second identifier ([[distributed-tracing]]).

## ELK Stack

The most common self-run log platform, named after its three parts:

| Part | Role |
|---|---|
| **Elasticsearch** | Stores and indexes log events; full-text and field search, aggregations |
| **Logstash** | Ingest pipeline: inputs → filters (parse, rename, enrich, drop) → outputs |
| **Kibana** | Search UI, dashboards, alert rules |

A fourth part, **Beats** (Filebeat for log files), is the lightweight shipper on each host; with it the product is called the Elastic Stack. A typical path:

```
application (stdout) → Filebeat or Fluent Bit → [Kafka buffer] → Logstash → Elasticsearch → Kibana
```

- With JSON logs most of Logstash's parsing work disappears; many setups ship straight from the collector to Elasticsearch.
- A buffer (Kafka) between shippers and the store is what keeps a slow or restarting Elasticsearch from losing logs or pushing back on applications.

Operating it:

- **Time-based indices and lifecycle policies**: new data on fast nodes, older data moved to cheaper tiers, then deleted. Retention is the main cost control.
- **Mapping explosion**: every distinct field name becomes an index field. Logging arbitrary maps (HTTP headers, user-supplied keys) as top-level fields creates thousands and degrades the cluster. Keep a fixed schema; put free-form data in one non-indexed field.
- **Sizing is storage-driven**: a full-text index is roughly the size of the raw data or more. This is the expensive way to keep logs.
- Variants and alternatives: **EFK** (Fluentd or Fluent Bit instead of Logstash, common on Kubernetes), **OpenSearch** (the open-source fork), **Grafana Loki** (indexes only labels and stores compressed lines — far cheaper, slower for free-text search), and hosted services (Splunk, Datadog, the cloud providers' log services).

## What this means for AEM

- **AEM logs are plain text, not JSON**: Sling's logging (Logback underneath) writes pattern-formatted lines to `error.log`, `request.log` and `access.log`. Any pipeline needs a parsing step, and multi-line stack traces must be joined by the shipper.
- **Levels are set per logger through OSGi configuration**, so DEBUG can be enabled for one package. On AEMaaCS the level is normally driven by an environment variable so that development environments log DEBUG and production stays at INFO or WARN.
- **AEMaaCS can forward its logs** (AEM, dispatcher and CDN) to a customer's own log platform; without forwarding, logs are downloaded or tailed through Cloud Manager ([aemaacs-build-deploy-debugging](../cloud-service/aemaacs-build-deploy-debugging.md)).
- The request ID that `request.log` prints on the request and response lines is AEM's built-in correlation between those two lines; it is not propagated to anything AEM calls.
