# Observability — logs, metrics, traces, SLOs

General system-design reference (not AEM-specific). Monitoring answers "is it broken in a way I predicted"; observability answers "why is it broken in a way I didn't" — the difference is whether you can ask new questions of existing telemetry without shipping new code.

## The three signals (and what each is for)

- **Metrics**: pre-aggregated numbers over time. Cheap to store, fast to query, the alerting backbone — but aggregation destroys the ability to ask "which request". **Cardinality is the cost model**: every label combination is a separate time series; a `user_id` or raw-URL label turns 10 series into 10 million and kills the TSDB. High-cardinality questions belong in traces/logs, not metrics.
- **Logs**: per-event records. **Structured (JSON/key-value) or they're write-only** — grep doesn't scale past the first incident. Rules that pay off: one canonical "wide event" per request (all context in a single line) beats 20 scattered lines; log levels mean something (ERROR = a human should eventually look, not "caught and continued"); sample noisy INFO at the source — log *volume* is a cost and an outage vector (logging pipeline saturates during incidents, exactly when needed).
- **Traces**: a request's tree of spans across services — the only signal that answers "where did the 3 seconds go" in a distributed system. Requires **context propagation** (W3C `traceparent` headers) through every hop — one service dropping the header splits the trace; queues need explicit propagation in message headers ([[async-systems]]). Sampling is mandatory at volume: head sampling (decide at ingress, cheap, blind) vs **tail sampling** (decide after completion — keep all errors and slow traces, sample the boring; costlier but what you actually want).
- Join key discipline: put `trace_id` in logs and exemplar links in metrics, so you can pivot metric spike → example traces → their logs. Three disconnected silos is the common failure of "we have all three".

## The metrics that matter

- **RED** per service: Rate, Errors, Duration — for request-driven systems. **USE** per resource: Utilization, Saturation, Errors — for infra. Four golden signals = RED + saturation.
- **Latency is a distribution, never an average**: the mean hides a bimodal disaster; track p50/p95/p99. Percentile math gotcha: **you can't average percentiles** across hosts/windows — aggregate histograms, compute percentiles from the merged histogram.
- Instrument at the **caller** as well as the server: the user experiences client-observed latency (includes connect, retries, queueing the server never sees).
- Per-dependency metrics mirror [[reliability-patterns]]: timeout rate, retry rate, breaker state — breaker-open is a leading indicator; error-rate alerts are trailing.

## Alerting

- **Alert on symptoms (SLO burn), not causes** (CPU high). Cause-based alerts page for things users never see and miss things users do see. Causes become dashboards for diagnosis, not pages.
- Every page must be **actionable and urgent**; everything else is a ticket. Alert fatigue is a reliability risk in itself — the ignored pager is how real outages run for hours.
- **Burn-rate alerting**: page when the error budget is being consumed N× faster than sustainable (e.g. 14.4× over 1h = fast burn, page; 3× over 6h = slow burn, ticket). This auto-scales urgency with impact and kills the static-threshold tuning treadmill.

## SLOs

- Chain: **SLI** (measured indicator, e.g. "fraction of requests < 300 ms and non-5xx, measured at the LB") → **SLO** (internal target, "99.9% over 30 days") → **SLA** (external contract with penalties; always looser than the SLO).
- The **error budget** (1 − SLO) is the point: 99.9%/30d = ~43 min of allowed failure, which converts reliability from a moral debate into a resource — budget healthy → ship fast; budget burned → freeze features, pay down reliability. Without the policy ("what we do when the budget is gone"), an SLO is a wall poster.
- Pick SLIs at the **user boundary** (edge/LB, synthetic journeys), few in number (2–4 per service), covering availability + latency; a critical flow (checkout) deserves its own. 100% is not a target — every extra nine costs ~10× and the user's own network is worse than four nines anyway.
- Watch for the gap between measured SLI and user truth: measuring at the server misses DNS/CDN/client failures; availability per-request masks "every user fails 0.1% of the time" vs "0.1% of users fail 100% of the time" — segment by user/tenant for the second ([[api-design]] per-client telemetry is also what makes API deprecation possible).

## Operational gotchas

- Telemetry has a **cost curve**: logs ≫ traces ≫ metrics per unit of insight at scale; teams routinely pay more for logging than compute. Sampling, retention tiers, and dropping never-queried data are engineering, not stinginess.
- Instrument **before** the incident: the telemetry you wish you had is the definition of unobservable. OpenTelemetry as the vendor-neutral instrumentation layer is now the default choice (instrument once, swap backends).
- Clock skew lies in cross-host log timelines; trust trace parent-child causality over timestamps.
- Dashboards rot: the incident-review question "what dashboard would have shown this in 1 minute" is the only dashboard-design method that works.

## AEM tie-ins

- AEMaaCS gives you logs (`aemerror` via Cloud Manager, GMT timestamps — [aemaacs-build-deploy-debugging](../cloud-service/aemaacs-build-deploy-debugging.md)), CDN logs with rule annotations ([cloud-manager-program-types](../cloud-service/cloud-manager-program-types.md) `waf=`), and New Relic APM on AMS ([cloud-manager-newrelic-users-api](../cloud-service/cloud-manager-newrelic-users-api.md)) — but no custom metrics pipeline; teams typically forward logs to their own stack.
- The "log gap" diagnostic (60-min silence as the *signature* of a failure mode, [aemaacs-build-deploy-debugging](../cloud-service/aemaacs-build-deploy-debugging.md)) is a reminder that absence of telemetry is itself a signal worth alerting on (heartbeats/deadman switches).
