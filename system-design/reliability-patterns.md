# Resilience patterns — keeping failures local

The unifying idea: **failures are normal; the design question is whether they stay local or cascade.** Most cascades are self-inflicted — by missing timeouts, unbounded retries, and unbounded queues.

## Timeouts

- **Every remote call needs an explicit timeout.** Library defaults are the bug: many HTTP/DB clients default to infinite or minutes — one hung dependency then pins a worker thread per request until the pool is exhausted and *your* service is down. The caller died of someone else's slowness.
- Two distinct timeouts: **connect** (short, ~1s — the network either works or doesn't) and **read/request** (sized to the dependency's real latency, e.g. p99 × safety factor — not a round number picked in a meeting).
- **Timeout budgets / deadline propagation**: the edge sets a total deadline (say 2s) and each hop passes the *remaining* budget down (gRPC does this natively). Without it, a downstream call happily computes for 30s on behalf of a client that gave up at 2 — pure wasted load, and inner timeouts longer than outer ones mean the outer layer never sees the inner error, just its own timeout.
- A timeout is an **ambiguous outcome** — the work may have completed. Everything after a timeout must assume "maybe it happened": idempotency keys, reconciliation (see [[api-design]]).

## Retries

- Core rules: retry only transient/retryable errors (timeouts, connection resets, `503`, `429`) and only idempotent operations; exponential backoff **with jitter**; a cap on attempts *and* on total elapsed time vs the caller's deadline. Queue-consumer retries and dead-letter handling are in [[async-systems]].
- **Why jitter**: plain exponential backoff keeps clients that failed together retrying together — the recovering dependency is hit by synchronized waves at 1 s, 2 s, 4 s. Randomizing the wait spreads them out. The variants:
  - **Full jitter** — `sleep = rand(0, min(cap, base × 2^attempt))`. The default choice: least total load on the dependency.
  - **Equal jitter** — half fixed, half random: `t/2 + rand(0, t/2)`. Guarantees a minimum wait.
  - **Decorrelated jitter** — `sleep = min(cap, rand(base, previous × 3))`. Grows from the previous wait rather than the attempt number.
- If the server says when to come back (`Retry-After`), obey it instead of the local schedule.
- **Jitter is not only for retries.** Anything periodic and fleet-wide synchronizes by itself: cache TTLs set at the same moment expire at the same moment, cron jobs fire on the minute, tokens issued at deploy time all refresh together. Add a random offset to each ([[caching]] stampede).
- **Retry amplification** is the cascade mechanism: 3 retries per layer across 3 layers = up to 27× load exactly when the bottom layer is weakest. Fixes: retry at **one** layer only (usually the edge or the client, not every hop), and **retry budgets** — e.g. retries may be ≤ 10% of recent requests; past that, fail fast. A retry budget converts a retry storm into a controlled degradation.
- Hedged requests (send a second copy after p95 latency, take the first answer) cut tail latency but only for idempotent reads, and raise load — budget them too.

## Circuit breakers

- State machine: **closed** (normal, counting failures) → **open** on threshold (error rate or consecutive failures over a window — fail *immediately*, no call made) → after a cooldown, **half-open** (let a few probes through) → success closes, failure reopens.
- Two jobs: protect the **caller** (fail in microseconds instead of burning a thread per doomed call) and protect the **callee** (a struggling service gets breathing room instead of a retry storm — the breaker is the circuit-level complement to backoff).
- Open-circuit behavior is a product decision, not an exception type: serve stale cache (`stale-if-error`, [[caching]]), a default value, a degraded page, or an honest error. "Throw" is just the default-worst choice.
- Tuning traps: per-endpoint (not per-host) breakers, minimum request volume before tripping (don't open on 2 failures of 2 requests), and **breakers + retries interact** — retries hammering a half-open breaker reopen it forever; the probe traffic must be excluded from retry logic.

## Bulkheads

- Named after a ship's watertight compartments: **partition resources so one failure can only flood its own compartment.** A breaker says "stop calling X"; a bulkhead says "even if X hangs, it can only eat its own share".
- What gets partitioned: thread pools and connection pools **per dependency** (the slow recommendation service cannot take the threads checkout needs), concurrency limits **per tenant or per caller** (one noisy customer cannot starve the rest), and whole instance groups or cells per customer segment.
- Two implementations: a **semaphore** (cap on concurrent calls, runs on the caller's thread — cheap, but cannot interrupt a hung call) and a **dedicated thread pool** (real isolation and a queue bound, at the cost of context switches and lost thread-local context).
- At the infrastructure level the same idea is separate pools for critical and background traffic, a separate database for analytics, and **cell-based architecture** — independent copies of the whole stack each serving a slice of customers, so a bad deploy or poison request takes out one cell.
- Traps: a bulkhead without a timeout only fills more slowly; the compartments' sizes add up, so ten pools of 20 threads is 200 threads; and a full compartment must **reject immediately**, not queue without bound.

## Fallbacks

- What a call returns when the real answer is unavailable (timeout, open breaker, full bulkhead). The options, roughly best to worst for the user:
  - **Stale value** — the last good response from a cache (`stale-if-error`, [[caching]]).
  - **Default or static value** — generic recommendations instead of personal ones, a default configuration.
  - **Alternative source** — a second provider, a replica, a coarser precomputed dataset.
  - **Defer** — accept the request, queue the work, confirm later.
  - **Omit** — leave the component out of the response.
  - **Fail fast** — an honest, quick error.
- **Choose fail-open or fail-closed deliberately.** A failing recommendation call may fail open (show nothing). A failing authorization or fraud check must fail closed (deny). Money and permissions do not get a default value.
- Traps:
  - **The fallback is the least-tested code in the system**, exercised only during an incident. Run it routinely (see [[chaos-engineering]]) or prefer a design where the normal path and the failure path are the same code.
  - **A fallback that leans on another dependency moves the overload there**: "cache is down, read the database" sends the full uncached load to a database sized for the cache-hit case, and a small outage becomes a large one.
  - **A silent fallback hides the outage.** Count every fallback served and alert on the rate; otherwise the system is degraded for days and nobody knows.

## Graceful degradation

- The product-level counterpart of a fallback: when part of the system is unavailable or overloaded, **the system does less instead of doing nothing.** A product page without reviews still sells the product.
- It starts with classifying every dependency as **critical** (the request cannot succeed without it) or **optional** (the response is poorer without it). An optional dependency that can fail the request has been wired as critical by mistake — the most common finding when this is first tested.
- Standard forms: drop optional features (personalization, recommendations, live counters), serve cached or static content, switch to **read-only mode** when the write path is down, reduce quality (fewer results, smaller images, lower refresh rate), and shed low-priority traffic first.
- **Kill switches / feature flags** let an operator turn expensive features off in seconds without a deploy. A **brownout** does it automatically: optional work is dimmed in proportion to load.
- Requirements: the degraded modes are decided in advance with the product owner, each is observable (a metric and a dashboard state, not a guess), and the user is told the truth where it matters ("prices may be out of date").


## Backpressure

- Definition: a system's ability to **push load resistance upstream** — slow consumer makes the producer slow down (bounded queues, TCP flow control, reactive streams demand) instead of buffering unboundedly or dying.
- The law underneath (Little's Law): queue length = arrival rate × time-in-system. If arrivals exceed service rate, the queue — and therefore latency — grows **without bound**. An unbounded queue doesn't absorb overload, it converts overload into unbounded latency plus an OOM, delivered later, during recovery (processing a backlog of requests whose clients are long gone).
- Therefore: **every queue bounded**, every pool bounded, and a decision at the bound: block the producer (sync backpressure), shed load (reject new work — `429`/`503` + `Retry-After`), or drop oldest/degrade. Rejecting cheaply at the edge is almost always right ([[api-design]] rate limiting is backpressure at the API boundary; LB health/readiness is backpressure at the infra boundary, [[scalability]]).
- **Load shedding order** is a product decision made in advance: shed expensive/optional work first (recommendations before checkout), by priority class, before the system chooses for you by falling over.
- **Metastable failure / GC-of-doom**: past saturation, goodput collapses below pre-overload levels (timeouts + retries + queue overhead feed on each other) and the system stays down even after load returns to normal — only shedding or a restart clears it. This is why "we'll autoscale out of it" fails: scale-out takes minutes, collapse takes seconds.

## Redundancy

- More than one of everything whose loss would stop the service, so that a failure removes capacity instead of function. The patterns above limit the damage of a failure; redundancy is what is left standing.
- **Sizing**: **N+1** survives one failure; **N+2** survives a failure *during* maintenance (the realistic case — one node is always being patched); **2N** is a full duplicate.
- **Active-active** (all copies serve traffic) versus **active-passive** (a standby takes over). Active-active exercises every copy continuously, so a broken one is noticed. A passive standby — cold, warm or hot by how fast it can take over — is only as good as its last failover test.
- **The arithmetic**: components in series multiply availabilities (three 99.9% hops ≈ 99.7%); redundant copies in parallel multiply *un*availabilities (two 99% copies ≈ 99.99%). The second formula **assumes independent failures**, and that assumption is where redundancy fails.
- **Correlated failures** defeat it: copies in the same zone, on the same deploy, reading the same bad config, sharing one expiring certificate or one dependency fail together. Spread copies across failure domains (instance → zone → region → provider) and stagger changes across them.
- **Headroom is part of the design**: with two zones each must be able to carry everything, so normal utilization stays under 50%; with three, under 66%. Capacity that has to be *acquired* during the failure (autoscaling into the surviving zone) depends on a control plane that is itself likely impaired — provisioning for the loss in advance is called **static stability** ([[autoscaling]]).
- Costs and traps: the failover mechanism is a new component that can fail or trigger falsely; two actives that lose sight of each other give split brain ([[data-storage]]); redundant data is not a backup, since a bad delete is replicated faithfully.
- **Hidden single points of failure** are rarely servers: DNS, the certificate, the load balancer, the config or secrets service, the deploy pipeline, the one person who knows the procedure.

## Putting it together (dependency call recipe)

Per remote dependency: connect+read timeout (from measured p99) → bounded retry with jitter, only on retryable errors, within the propagated deadline, under a retry budget → circuit breaker with a chosen fallback → bulkheaded pool → metrics on timeouts/retries/breaker-state/fallbacks served (see [[observability]] — breaker state changes are *the* leading incident indicator). Then **test it** by injecting the failure on purpose — [[chaos-engineering]]; an untested fallback is a second bug waiting behind the first.

In Java these are library features rather than hand-written code: Resilience4j provides time limiter, retry, circuit breaker, bulkhead (semaphore and thread-pool) and rate limiter as composable decorators; Hystrix, the library that popularized the patterns, is in maintenance mode.

## AEM tie-ins

- Blocked distribution queues are head-of-line blocking without a DLQ-replay path — the triage doc is [content-distribution-journal](../cloud-service/content-distribution-journal.md).
- Adobe I/O Runtime's 60s blocking-call cap is a platform-enforced timeout budget: "long-running" work must become fire-and-forget ([app-builder](../cloud-service/app-builder.md)).
- CDN `stale-if-error` and Dispatcher serving cached content while publish is down are circuit-breaker-shaped degradation for content delivery ([[caching]]).
