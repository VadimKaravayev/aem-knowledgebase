# Scalability — horizontal scaling, stateless services, load balancers

General system-design reference (not AEM-specific). AEMaaCS's publish tier is a textbook horizontal-scaling case study — tie-ins at the end.

## Vertical vs horizontal

- **Vertical (scale up)**: bigger box. Zero architecture change, but a hard ceiling, a single point of failure, and cost grows super-linearly at the top end. Still the right first move for databases and anything stateful — don't shard what a bigger instance solves.
- **Horizontal (scale out)**: more boxes behind a balancer. Near-linear capacity *if and only if* nodes don't coordinate — every shared lock, shared cache write, or chatty cross-node call bends the curve (Amdahl: the serial fraction caps speedup; USL adds the coordination penalty that makes throughput *decrease* past a point).
- The real question is never "can we add nodes" but "**what state stops us**" — scaling is a state-placement problem wearing a capacity costume.

## Stateless services

- Stateless = any instance can serve any request, because no request depends on instance-local memory/disk surviving between requests. This is what makes horizontal scaling, rolling deploys, autoscaling and instance replacement *safe* — the scheduler can kill any pod any time.
- Where the state goes instead: sessions → signed cookie/JWT or Redis; files → object storage, never local disk; in-memory caches → fine, but only as rebuildable L1 (correctness must not depend on them); scheduled jobs → a distributed scheduler or leader election, or N instances run the job N times (the classic "why did the email send 6 times" incident).
- **Sticky sessions are a confession**, not a solution: they make the LB compensate for statefulness, break on instance death (users logged out on every deploy), and skew load. Acceptable as a migration crutch; design target is no stickiness.
- Warm-container gotcha: "stateless" platforms often have *apparently* persistent local state on warm instances (works in testing, vanishes under scale-out) — never read back what you wrote locally. Same trap documented for Adobe I/O Runtime in [app-builder](../cloud-service/app-builder.md).

## Load balancers

- **L4 (transport)**: routes TCP/UDP connections; fast, protocol-blind, no per-request decisions. **L7 (application)**: terminates HTTP(S), routes per request — path/header/cookie rules, retries, TLS offload, per-request metrics. Most app traffic wants L7; L4 for raw throughput, non-HTTP, or when TLS must pass through.
- Algorithms: round-robin (default, assumes uniform requests), **least-connections** (better under heterogeneous request cost), weighted (canaries, mixed hardware), consistent hashing (cache-affinity routing — key insight: it minimizes remapping when nodes change, the same trick as sharded caches).
- **Health checks are the LB's eyes**: distinguish *liveness* (process up) from *readiness* (able to serve — dependencies warm, not overloaded). A readiness check that calls a shared dependency (the DB) takes the **whole fleet** out when that dependency blips — check your own ability to serve, not your dependencies' uptime. Failing readiness during overload sheds load; failing liveness during overload causes restart storms.
- Connection draining on deploys (stop new, finish in-flight) is what makes "rolling restart" invisible; without it, every deploy is a micro-outage.
- LBs are also the enforcement point for timeouts, request size caps and rate limits — the edge is where rejecting is cheapest ([[api-design]], [[reliability-patterns]]).

## Autoscaling

- Scale on a metric that *leads* saturation: CPU works for CPU-bound services; for I/O-bound ones use concurrency/queue depth/latency. Scaling on latency alone oscillates (latency reacts late and recovers late).
- Scale-out must be faster than your traffic ramp (instance boot + warmup counts); scale-in must be slower than your traffic dips (flapping). Set min capacity to survive a cold cache + one AZ loss.
- Autoscaling doesn't fix a bottleneck *behind* the scaled tier: doubling app servers against a fixed DB just moves the queue — and can turn a slowdown into a DB outage (more connections, more pressure). Scale the constraint, not the symptom.
- Databases scale differently: read replicas for reads (with staleness consequences — see [[data-storage]]), sharding for writes (last resort: cross-shard queries/transactions get hard), connection pooling always (per-instance pools × many instances = DB connection exhaustion; use a proxy/pooler like PgBouncer).

## AEM tie-ins

- AEMaaCS publish = the pattern executed: a farm of pods, each with its own repo + 1:1 dispatcher, scaled horizontally; new pods catch up by replaying the journal from their own offset — state convergence without coordination ([aemaacs-architecture-overview](../cloud-service/aemaacs-architecture-overview.md), [content-distribution-journal](../cloud-service/content-distribution-journal.md)).
- The author tier shows the stateful counter-case: TarMK can't cluster → "multi-author" means partitioning, not load balancing; only MongoMK gets an LB ([multi-author-scaling](../infrastructure-ops/multi-author-scaling.md)).
- AMS vs AEMaaCS is the vertical/fixed vs horizontal/elastic exam contrast — Cloud Manager alone gives AMS no elasticity ([ams-vs-aemaacs](../cloud-service/ams-vs-aemaacs.md)).
