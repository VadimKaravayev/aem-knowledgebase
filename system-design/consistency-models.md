# Consistency models — from strong to eventual

What a replicated system promises a reader about which writes it will see, and in what order. Docs-based. Replication mechanics, CAP/PACELC and quorums live in [[data-storage]]; transaction isolation (a different axis — see the last section) lives in [[database-transactions-isolation]].

## The ladder

Each model allows everything the one above it allows, plus more anomalies — and costs less coordination.

| Model | Promise | Still allowed | Survives a partition and stays available? |
|---|---|---|---|
| **Strong** (linearizable) | One copy of the data, as far as anyone can tell; a completed write is visible to every later read | Nothing surprising | No — the minority side must refuse |
| **Sequential** | Everyone sees the same single order of operations, and each client's own operations keep their order | That order may lag real time: a read can miss a write that already completed elsewhere | No |
| **Causal** | An effect is never seen before its cause | Unrelated (concurrent) writes seen in different orders on different replicas | Yes, if clients stick to a replica |
| **Monotonic reads** (a session guarantee) | A client never sees time go backwards | Stale data; missing your own write | Yes, if clients stick to a replica |
| **Eventual** | Replicas converge once writes stop | Anything in the meantime | Yes |

**Sticky sessions** are not a rung on the ladder: they are a routing trick that buys the session-level rungs cheaply.

## Strong consistency (linearizability)

- Every operation appears to take effect **atomically at one instant between its request and its response**, and that order matches real time. Consequence: as soon as a write is acknowledged, *every* client, on *every* replica, reads it — and once one reader has seen the new value, no later reader sees the old one.
- Cost: a write (and usually a read) must coordinate with a majority — consensus (Raft, Paxos) or synchronous replication. That means latency of the slowest needed replica, and **unavailability on the minority side of a partition** (the "C" choice in CAP).
- Needed where a stale read breaks an invariant rather than a screen: locks and leases, leader election, uniqueness (usernames, seat booking), "balance never below zero", compare-and-set.
- Traps:
  - **`W + R > N` is not linearizable by itself**: concurrent writes, sloppy quorums and a write that fails halfway (visible to some readers, then not) all break it. It needs read repair before returning, or consensus.
  - **Reading from the leader is not automatically linearizable**: a deposed leader that has not noticed yet serves stale reads. Real systems confirm leadership per read (quorum round-trip, lease with bounded clock drift).
  - "Strong consistency" in a vendor's docs often means only "reads from the primary" or "read-your-writes". Ask which anomaly is excluded.

## Sequential consistency

- There is **one total order** of all operations that every node agrees on, and it preserves each client's own program order — but it **need not match wall-clock time** across clients.
- Difference from linearizable, in one example: A's write completes; B reads a second later and gets the old value. Linearizable: forbidden. Sequential: fine, as long as the system can explain it by ordering B's read before A's write, and nobody ever sees the two in the opposite order.
- What it gives: all replicas apply the same log in the same order, so nobody disagrees about history — they only disagree about how far along they are. This is what a **follower replaying an ordered log** provides.
- Familiar example: ZooKeeper — writes are linearizable, reads are served locally by any server and may lag, so the overall guarantee for reads is sequential (a `sync` before the read narrows the gap).
- Trap: it is not composable. Two objects that are each sequentially consistent do not make a sequentially consistent pair; linearizability does compose. This is why coordination services aim for the stronger model.

## Causal consistency

- Operations that are **causally related** — one could have influenced the other (same client's sequence, or a write made after reading another write) — are seen by everyone in that order. **Concurrent** operations have no required order, and replicas may disagree on it.
- The user-visible meaning: a reply is never shown before the message it answers; an "unshare, then post" is never applied as "post, then unshare".
- It is a **partial order**, not a total one, so it needs no global coordination. It is the strongest model that can stay available during a partition — provided each client keeps talking to the same replica.
- Implementation: every write carries its dependencies (version vectors, or explicit dependency lists); a replica **delays applying a write until its dependencies have arrived**. Cost is metadata that grows with the number of writers, and it only covers causality the system can see — a dependency created outside it (a phone call, another service) is invisible.
- Concurrent writes to the same item still need conflict resolution (see eventual consistency below); "causal+" is the name for causal plus guaranteed convergence.

## Monotonic reads

- **Once a client has seen a version, it never sees an older one.** No promise of freshness — only no going backwards.
- The anomaly it removes: refresh shows a new comment, refresh again and it is gone, a third refresh brings it back. Cause: consecutive reads hit replicas with different lag.
- Two implementations:
  - **Sticky routing** — the same client always reads the same replica (next section).
  - **Version token** — the client carries the highest version (LSN, timestamp, vector) it has seen; a replica behind that either waits, or the read is redirected. Survives failover, which stickiness does not.
- It is one of four **session guarantees**; name the one you need rather than asking for "strong":
  - **Read-your-writes** — after my write, my reads include it. *Not* implied by monotonic reads: I can be pinned to a replica that has not received my write yet.
  - **Monotonic reads** — as above.
  - **Monotonic writes** — my writes are applied in the order I issued them.
  - **Writes-follow-reads** — a write I make after reading X is ordered after X everywhere.
- All four together are roughly what causal consistency gives a single session.

## Sticky sessions

- A **load-balancer mechanism, not a consistency model**: route the same client to the same backend, via an affinity cookie or a hash of a stable key.
- What it buys on a replicated read tier: **monotonic reads** for free (one replica never goes backwards), and **read-your-writes** only if that same node also took the write, or the writer is pinned to the leader for a short window after writing.
- Where it breaks — exactly when it matters:
  - **The pinned node dies, is drained or scaled in**: the client lands on a different replica, possibly further behind. The guarantee holds in steady state and fails during incidents and deploys.
  - **Hashing by connection or cookie misses the second device**: hash on user ID if the guarantee must hold across phone and laptop.
  - **Load skew**: a few heavy clients pin to one node; autoscaling adds capacity that existing sessions never use.
- This is the consistency-side reason stickiness exists; [[scalability]] covers why it is a liability for stateless design. The two meet at the same rule: use stickiness as an optimisation, and carry a version token (or keep session state outside the node) for correctness.

## Eventual consistency

- **If writes stop, all replicas eventually hold the same value.** That is a liveness promise only: it says nothing about how long, nor what is readable in between — stale values, values going backwards, your own write missing.
- Where it is the right answer: DNS, CDN and cache layers, asynchronous read replicas, counters and feeds, anything where availability and latency beat freshness and a stale read costs a refresh, not money.
- Convergence needs two mechanisms the model itself does not specify:
  - **Anti-entropy** — how missed writes arrive: read repair, hinted handoff, background comparison (Merkle trees), or replaying a log.
  - **Conflict resolution** — what happens to concurrent writes: **last-write-wins silently discards one** (and clock skew picks the loser); alternatives are version vectors surfacing siblings to the application, or CRDTs.
- **Strong eventual consistency** (CRDTs): replicas that have received the same set of updates are in the same state, with no conflict step — convergence without coordination, at the price of restricting data types to mergeable ones.
- Design consequence: the window is measured, not assumed. Track replication lag as a first-class metric, and decide per read whether it may be stale — most systems mix models (strong for the uniqueness check, eventual for the listing).

## Consistency is not isolation

- These models are about **replicas of one object** agreeing. Isolation levels are about **concurrent multi-object transactions** not interfering. Serializable says transactions behave as if run one at a time, in *some* order — not necessarily real-time order; linearizable is real-time but single-object. Both together is **strict serializability**.
- The "C" in ACID is a third thing again: application invariants holding. The "C" in CAP is linearizability.

## What this means for AEM

- **Publish tier is eventually consistent, with a sequential flavour**: each publish instance has its own repository and applies the same ordered distribution log at its own pace. Consecutive requests can hit instances at different points in that log — a non-monotonic read for the visitor, normally hidden by the dispatcher and CDN caches ([content-distribution-journal](../cloud-service/content-distribution-journal.md)).
- **Author on a document store is a cluster with visible lag**: Oak cluster nodes pick up each other's changes on a background read (about a second by default), so a write on one node followed immediately by a read on another can miss it. This is why AEM 6.5 MongoMK author clusters are run with sticky sessions on the load balancer — stickiness supplying read-your-writes. Irrelevant on AEMaaCS, where author routing is Adobe-managed.
- **Publish logins on AEMaaCS**: cookie-based stickiness to a publish pod exists but is best-effort (lost on scaling and pod replacement), so per-user state must not depend on it — encapsulated tokens and user-data synchronisation are the correctness mechanism, stickiness only the optimisation.
