# Data — SQL vs NoSQL, indexes, replication, consistency

General system-design reference (not AEM-specific). Oak/AEM instances of each concept linked at the end.

## SQL vs NoSQL (the honest version)

- The real axes are not "relational vs modern" but: **query flexibility** (ad-hoc joins vs access-pattern-first design), **consistency model** (ACID transactions vs tunable/eventual), **scaling story** (vertical + read replicas + manual shard vs native partition), and **schema enforcement point** (write time vs read time — "schemaless" means every reader carries the schema).
- Default to a relational DB (Postgres) until a specific limit is hit — modern SQL handles JSON columns, full-text, ~TB scale. NoSQL categories and when they earn it: **key-value** (Redis/Dynamo — known-key access at extreme scale/latency), **document** (Mongo — aggregate-shaped data, few cross-aggregate queries), **wide-column** (Cassandra — write-heavy time-series at scale, query patterns known upfront), **graph** (deep traversals that kill recursive SQL), **search** (Elasticsearch — inverted-index queries; it's a *secondary* index, not a system of record).
- Dynamo/Cassandra-style design is **access-pattern-first**: you denormalize into the shapes your queries need and maintain them on write. Picking NoSQL *then* discovering a new query pattern is the standard regret; relational buys you the right to not know your queries yet.
- Polyglot rule: every extra store adds a consistency boundary you now own (cache-invalidation-grade problems between DB and search index — usually solved with CDC, see outbox in [[async-systems]]).

## Indexes

- A B-tree index = sorted structure → O(log n) point/range lookups on the **leftmost prefix** of the indexed columns: index `(a, b)` serves `WHERE a=?` and `WHERE a=? AND b>?` but **not** `WHERE b=?`. Column order in composite indexes is the design decision: equality columns first, then the range column; a range on `a` stops `b` from being used for seeking.
- Every index taxes writes (each INSERT/UPDATE maintains all of them) and space; "index every column" makes writes crawl. Index for the queries you actually run — read the query plan (`EXPLAIN`), don't guess.
- Indexes go stale as data distribution shifts (planner statistics); a query that was fast at 1M rows flips to a sequential scan at 100M. Also: functions on the indexed column (`WHERE lower(email)=?`), leading-wildcard LIKE, and implicit type casts all silently bypass the index.
- **Covering index** (all selected columns in the index) skips the table lookup entirely — the cheapest big win for hot queries. **Partial/filtered index** (`WHERE deleted = false`) keeps the index small and matches the real query.
- Low-cardinality columns (status flags) index poorly alone but fine as a composite prefix for equality.
- Oak instance: property vs Lucene index = sync-exact vs async-inverted, with the async-lag accuracy gotcha and config-as-code `/oak:index` rules — [oak-indexing-aemaacs](../cloud-service/oak-indexing-aemaacs.md). Reindex time bounding deploy duration ([rolling-deployment-two-version-overlap](../cloud-service/rolling-deployment-two-version-overlap.md)) is the "indexes are write-time cost" lesson at platform scale.

## Replication

- **Why**: read scaling, availability, geo-latency, zero-downtime maintenance — each wants a different topology.
- **Single-leader** (default): writes → leader, replicated to followers. **Async** replication = low write latency, but follower lag → a failover can **lose acknowledged writes**, and reads from replicas are stale. **Sync** = no loss but write latency = slowest replica and availability coupled to it; semi-sync (one sync follower) is the common compromise.
- **Read-your-own-writes** is the classic lag bug: user saves, next page reads a replica, edit "disappears". Fixes: pin the writing user to the leader briefly, session token with min-LSN ("read at least this version"), or just read-after-write from the leader.
- **Failover** is where replication designs die: split brain (two leaders — fence the old one), picking the most-caught-up replica, clients re-resolving. "We have a replica" ≠ "we have tested failover".
- **Multi-leader / leaderless** (Dynamo-style quorums, `W + R > N`): write availability across regions at the price of **conflict resolution** — last-write-wins silently drops data (clock skew decides the winner); real answers are CRDTs or app-level merge. Avoid multi-leader unless geography forces it.
- Replication ≠ backup: replicas faithfully replicate your `DELETE`. (Same lesson as TarMK cold standby: "no integrity checking, corruption replicates faithfully — availability, not backup" — [tarmk-cold-standby](../infrastructure-ops/tarmk-cold-standby.md).)

## Consistency

- **CAP, stated correctly**: only *during a network partition* you choose consistency (refuse) or availability (serve possibly-stale). No partition → no trade-off; the everyday trade-off is **PACELC**'s "else": latency vs consistency.
- Spectrum: **linearizable** (one global order — expensive, needed for uniqueness/locks/leader election) → **sequential/causal** (causes precede effects; what users intuitively expect) → **eventual** (converges, no promise when). Most systems are consistent per-entity (single-row atomicity, single-partition ordering) and eventual across entities — design aggregates so invariants live *inside* one atomic unit.
- Session guarantees (read-your-writes, monotonic reads) are the cheap, user-visible subset — ask for these by name instead of "strong consistency everywhere".
- Cross-service "transactions": 2PC is a blocking availability hazard; the practical pattern is **sagas** — a sequence of local transactions with compensating actions, which means intermediate states are *visible* and the business flow must tolerate them ([[async-systems]] outbox/idempotency are the building blocks).
- AEM instance: every publish pod is an independently-converging replica fed by an ordered log, and a content package mixing mutable/immutable halves installs non-atomically — eventual consistency and partial-visibility, in CMS clothing ([content-distribution-journal](../cloud-service/content-distribution-journal.md), [rolling-deployment-two-version-overlap](../cloud-service/rolling-deployment-two-version-overlap.md)).

## Design-review checklist

Named consistency need per data class (linearizable / causal / eventual + staleness budget); composite-index column order justified by the query list; replica lag monitored and surfaced to the app (not just ops); failover *tested*; conflict policy chosen explicitly (never default LWW); every denormalized copy has a documented repair path (backfill/CDC).
