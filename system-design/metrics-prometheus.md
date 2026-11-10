# Metrics and Prometheus — types, RED and USE, exporters

Numbers over time: which kinds exist, which ones to collect, and how Prometheus gathers and stores them. Docs-based. Overview of all three signals: [[observability]]; siblings: [[logging]], [[distributed-tracing]], [[slo-error-budgets-alerting]].

## Metric types

| Type | Behaviour | Use for | Query with |
|---|---|---|---|
| **Counter** | Only goes up; resets to zero on restart | Requests served, errors, bytes sent | `rate()` — the per-second increase |
| **Gauge** | Goes up and down; a current value | Queue depth, memory in use, open connections, temperature | The value itself, `avg`, `max` |
| **Histogram** | Counts observations into buckets, plus a running sum and count | Request duration, response size | `histogram_quantile()` for percentiles |
| **Timer** | A duration measurement: how many, how long in total, the longest, optionally the distribution | Any timed operation | Becomes a histogram or summary in the backend |

- A **counter's raw value is meaningless** — it depends on when the process started. What matters is its rate. Never use a gauge for something that only accumulates, or a counter for something that can fall.
- A **histogram is the only type that gives correct percentiles across many instances**, because bucket counts can be added together. The price is one time series per bucket, and precision limited by where the bucket boundaries sit — put boundaries around the latency target.
- A **summary** computes percentiles inside the application. Cheap to query, but **percentiles from different instances cannot be combined**; averaging them is simply wrong. Prefer histograms.
- **Timer** is the vocabulary of instrumentation libraries (Micrometer, Dropwizard Metrics) rather than a Prometheus type: recording one timer produces a count, a total time and a maximum, and a histogram when percentiles are enabled.
- **Averages hide the problem.** A mean latency of 200 ms can be every request at 200 ms, or 95% at 50 ms and 5% at 3 seconds. Track p50, p95 and p99.

## What to measure

### RED — for every service

| | Question | Typical metric |
|---|---|---|
| **R**ate | How many requests per second? | `rate(http_requests_total[5m])` |
| **E**rrors | How many of them fail? | The same counter filtered by status, divided by the total |
| **D**uration | How long do they take? | A latency histogram, read as p50/p95/p99 |

RED describes what the **user of the service** experiences. It is the same three questions for every request-driven service, which is what makes a standard dashboard possible.

### USE — for every resource

| | Question | Example for CPU / disk / pool |
|---|---|---|
| **U**tilization | How busy is it? | CPU 85% / disk 70% full / 18 of 20 connections in use |
| **S**aturation | How much work is waiting? | Run-queue length / I/O queue / threads waiting for a connection |
| **E**rrors | Is it failing? | Machine-check errors / I/O errors / connection timeouts |

USE describes the **machinery**: CPU, memory, disk, network, and software resources such as thread and connection pools. Saturation is the one most often missing and the one that explains latency — a resource at 100% utilization with no queue is fine, one at 70% with a queue is not.

RED tells you *that* users are affected; USE helps find *why*. The **four golden signals** (latency, traffic, errors, saturation) are RED plus saturation.

## Prometheus

An open-source monitoring system: it collects metrics, stores them in its own time-series database, and evaluates queries and alert rules over them.

### How it works

- **Pull model.** Each application exposes its current metric values as text on an HTTP endpoint (`/metrics`). Prometheus **scrapes** every target on an interval (15–60 seconds is typical) and stores what it reads.
- **Service discovery** finds the targets — Kubernetes, cloud APIs, Consul, or a file — so nothing is registered by hand.
- Every scrape also records **`up`** for the target: 1 if the scrape worked, 0 if not. A free liveness signal for everything monitored.
- **Labels** make metrics multi-dimensional: `http_requests_total{service="orders", method="POST", status="500"}`. Each distinct combination is its own time series.
- **PromQL** queries and aggregates them; **Alertmanager**, a separate component, receives firing alerts and handles grouping, deduplication, silencing and routing to people. **Grafana** is the usual dashboard layer on top.

```
# What an application exposes
http_requests_total{method="POST",status="200"} 10432
http_request_duration_seconds_bucket{le="0.1"} 9120
http_request_duration_seconds_bucket{le="0.5"} 10301
http_request_duration_seconds_bucket{le="+Inf"} 10432
http_request_duration_seconds_sum 1187.4
http_request_duration_seconds_count 10432
```

```promql
# Rate: requests per second, per service
sum by (service) (rate(http_requests_total[5m]))

# Errors: share of requests that returned 5xx
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# Duration: 99th percentile across all instances
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
```

### Traps

- **Cardinality is the cost.** A label whose values are unbounded — user ID, order ID, full URL, error message — creates a time series per value and exhausts memory. Labels are for small fixed sets (method, status class, route template). Per-request detail belongs in logs and traces.
- **`rate()` needs a window several times the scrape interval**, or it has too few samples and returns nothing.
- **Short-lived jobs** end before they are scraped. The Pushgateway exists for that case only; using it as a general push channel leaves stale values behind forever.
- **A single Prometheus is not highly available and not long-term storage.** Local retention is days to weeks. Run two identical servers for availability, and send data to a system built for long retention and a global view (Thanos, Cortex, Grafana Mimir, VictoriaMetrics) when that is needed.
- **Naming**: base units (`_seconds`, `_bytes`), `_total` on counters. A metric in milliseconds next to one in seconds causes wrong dashboards.

## TSDB — time-series database

A database specialized for **(series, timestamp, value)** data, where a series is a metric name plus its labels.

- The workload it is built for: almost only appends, in time order; reads are ranges over recent time; old data is deleted in bulk or downsampled, never updated.
- That allows **heavy compression** — timestamps arrive at regular intervals and consecutive values change little, so a sample costs a byte or two instead of sixteen.
- An **inverted index on labels** finds the series matching a query; data is stored in **time blocks**, so dropping old data is deleting a block.
- In Prometheus: recent samples sit in memory backed by a write-ahead log, are cut into two-hour blocks on disk, and blocks are compacted into larger ones.
- Why not a relational database: a row and a B-tree entry per sample is far larger and slower at this volume. Why not a log store: metrics are numeric and regular, and indexing them as text wastes both properties.
- Others: InfluxDB, TimescaleDB (a PostgreSQL extension), VictoriaMetrics.

## Exporters

An **exporter** is a small process that reads statistics from something that does not speak Prometheus and exposes them on a `/metrics` endpoint. Your own services are instrumented in code; everything else gets an exporter.

| Exporter | Reads | Key metrics | Default port |
|---|---|---|---|
| **Node exporter** | The host's operating system (Linux and other Unix) | CPU by mode, memory, disk space and I/O, filesystem, network, load | 9100 |
| **Blackbox exporter** | Nothing internal — it **probes an endpoint from outside** over HTTP, HTTPS, DNS, TCP, ICMP or gRPC | `probe_success`, probe duration, HTTP status, TLS certificate expiry | 9115 |
| **MySQL exporter** (`mysqld_exporter`) | Server status and schema statistics | Connections, queries and slow queries, InnoDB buffer pool, replication lag | 9104 |
| **PostgreSQL exporter** (`postgres_exporter`) | The `pg_stat_*` views | Connections against the limit, transactions, locks, dead rows, replication lag | 9187 |

- **Node exporter** is the USE method for a machine. One per host (a DaemonSet on Kubernetes). It reports the host, not the applications on it.
- **Blackbox exporter** is the outside view: it answers "can a client reach this and get a correct answer", which internal metrics cannot. Prometheus passes the target to probe as a parameter, so one exporter checks many endpoints. It is also the simplest way to alert on **expiring certificates**.
- **Database exporters** connect as a database user with monitoring rights only. Give them a dedicated low-privilege account, and watch replication lag and connection saturation first — those two precede most database incidents.
- Others in the same family: JMX exporter (JVM and anything exposing MBeans), cAdvisor (containers), kube-state-metrics (Kubernetes objects).

**White-box and black-box** monitoring are both needed: internal metrics explain behaviour, external probes prove reachability. A service can report perfect internal health while DNS or a certificate keeps every user out.

## What this means for AEM

- **AEMaaCS exposes no metrics endpoint to scrape**; there is no place to run an exporter. What is available is what Adobe surfaces, plus anything derived from forwarded logs. Customer-side metrics in practice are **black-box probes** against the published site and CDN.
- **AEM 6.5 that you operate** publishes its internals as JMX MBeans (repository statistics, replication queues, Sling job queues), so the JMX exporter plus a node exporter on each instance gives a workable Prometheus setup.
