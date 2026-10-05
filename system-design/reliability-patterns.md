# Reliability — timeouts, retries, circuit breakers, backpressure

General system-design reference (not AEM-specific). The unifying idea: **failures are normal; the design question is whether they stay local or cascade.** Most cascades are self-inflicted — by missing timeouts, unbounded retries, and unbounded queues.

## Timeouts

- **Every remote call needs an explicit timeout.** Library defaults are the bug: many HTTP/DB clients default to infinite or minutes — one hung dependency then pins a worker thread per request until the pool is exhausted and *your* service is down. The caller died of someone else's slowness.
- Two distinct timeouts: **connect** (short, ~1s — the network either works or doesn't) and **read/request** (sized to the dependency's real latency, e.g. p99 × safety factor — not a round number picked in a meeting).
- **Timeout budgets / deadline propagation**: the edge sets a total deadline (say 2s) and each hop passes the *remaining* budget down (gRPC does this natively). Without it, a downstream call happily computes for 30s on behalf of a client that gave up at 2 — pure wasted load, and inner timeouts longer than outer ones mean the outer layer never sees the inner error, just its own timeout.
- A timeout is an **ambiguous outcome** — the work may have completed. Everything after a timeout must assume "maybe it happened": idempotency keys, reconciliation (see [[api-design]]).

## Retries

- Core rules (detail in [[async-systems]]): retry only transient/retryable errors, exponential backoff **with jitter**, a cap on attempts *and* on total elapsed time vs the caller's deadline.
- **Retry amplification** is the cascade mechanism: 3 retries per layer across 3 layers = up to 27× load exactly when the bottom layer is weakest. Fixes: retry at **one** layer only (usually the edge or the client, not every hop), and **retry budgets** — e.g. retries may be ≤ 10% of recent requests; past that, fail fast. A retry budget converts a retry storm into a controlled degradation.
- Hedged requests (send a second copy after p95 latency, take the first answer) cut tail latency but only for idempotent reads, and raise load — budget them too.

## Circuit breakers

- State machine: **closed** (normal, counting failures) → **open** on threshold (error rate or consecutive failures over a window — fail *immediately*, no call made) → after a cooldown, **half-open** (let a few probes through) → success closes, failure reopens.
- Two jobs: protect the **caller** (fail in microseconds instead of burning a thread per doomed call) and protect the **callee** (a struggling service gets breathing room instead of a retry storm — the breaker is the circuit-level complement to backoff).
- Open-circuit behavior is a product decision, not an exception type: serve stale cache (`stale-if-error`, [[caching]]), a default value, a degraded page, or an honest error. "Throw" is just the default-worst choice.
- Tuning traps: per-endpoint (not per-host) breakers, minimum request volume before tripping (don't open on 2 failures of 2 requests), and **breakers + retries interact** — retries hammering a half-open breaker reopen it forever; the probe traffic must be excluded from retry logic.
- Bulkheads are the sibling pattern: partition resources (thread pools, connection pools, instance groups) per dependency/tenant so one dependency's hang can't drain the shared pool. A breaker says "stop calling X"; a bulkhead says "even if X hangs, it can only eat its own compartment".

## Backpressure

- Definition: a system's ability to **push load resistance upstream** — slow consumer makes the producer slow down (bounded queues, TCP flow control, reactive streams demand) instead of buffering unboundedly or dying.
- The law underneath (Little's Law): queue length = arrival rate × time-in-system. If arrivals exceed service rate, the queue — and therefore latency — grows **without bound**. An unbounded queue doesn't absorb overload, it converts overload into unbounded latency plus an OOM, delivered later, during recovery (processing a backlog of requests whose clients are long gone).
- Therefore: **every queue bounded**, every pool bounded, and a decision at the bound: block the producer (sync backpressure), shed load (reject new work — `429`/`503` + `Retry-After`), or drop oldest/degrade. Rejecting cheaply at the edge is almost always right ([[api-design]] rate limiting is backpressure at the API boundary; LB health/readiness is backpressure at the infra boundary, [[scalability]]).
- **Load shedding order** is a product decision made in advance: shed expensive/optional work first (recommendations before checkout), by priority class, before the system chooses for you by falling over.
- **Metastable failure / GC-of-doom**: past saturation, goodput collapses below pre-overload levels (timeouts + retries + queue overhead feed on each other) and the system stays down even after load returns to normal — only shedding or a restart clears it. This is why "we'll autoscale out of it" fails: scale-out takes minutes, collapse takes seconds.

## Putting it together (dependency call recipe)

Per remote dependency: connect+read timeout (from measured p99) → bounded retry with jitter, only on retryable errors, within the propagated deadline, under a retry budget → circuit breaker with a chosen degraded response → bulkheaded pool → metrics on timeouts/retries/breaker-state (see [[observability]] — breaker state changes are *the* leading incident indicator). Then **test it**: fault injection (kill the dependency, add latency) in staging or game days; an untested fallback is a second bug waiting behind the first.

## AEM tie-ins

- Blocked distribution queues are head-of-line blocking without a DLQ-replay path — the triage doc is [content-distribution-journal](../cloud-service/content-distribution-journal.md).
- Adobe I/O Runtime's 60s blocking-call cap is a platform-enforced timeout budget: "long-running" work must become fire-and-forget ([app-builder](../cloud-service/app-builder.md)).
- CDN `stale-if-error` and Dispatcher serving cached content while publish is down are circuit-breaker-shaped degradation for content delivery ([[caching]]).
