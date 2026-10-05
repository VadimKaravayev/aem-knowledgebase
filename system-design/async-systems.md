# Async systems — queues, events, Kafka concepts, retries, DLQs

General system-design reference (not AEM-specific). AEMaaCS's own publication pipeline is a Kafka-based worked example — see the tie-in at the end.

## Queues vs events (the two messaging intents)

- **Command/work queue** (SQS, RabbitMQ): one consumer *should* process each message; the message is a task; deleting after processing is the point. Scaling = more consumers competing on one queue.
- **Event stream/log** (Kafka, Kinesis): the message is a fact ("order placed"); **many independent consumer groups** each read the whole stream at their own pace; retention is time/size-based, not consumption-based — consuming doesn't delete.
- Design smell: an "event" whose name is an imperative ("SendEmail") is a command in disguise — it couples the producer to one specific consumer. Events describe what happened; commands tell someone what to do. Mixing them muddles ownership of retries and failure.

## Kafka concepts (the five that matter)

- **Topic → partitions**: a topic is split into partitions; **ordering is guaranteed only within a partition**, and the partition is chosen by the message key (same key → same partition → ordered). Total order across a topic requires 1 partition — which caps throughput to one consumer; this trade-off is the whole game.
- **Consumer groups**: each partition is owned by exactly one consumer *within a group* at a time → max useful parallelism = partition count (over-provisioned consumers sit idle). Choose partition count up front; repartitioning breaks key→partition mapping and thus ordering history.
- **Offsets**: consumer position per partition, committed by the consumer. Commit *after* processing = **at-least-once** (duplicates on crash); commit *before* = at-most-once (loss). "Exactly-once" in practice = at-least-once delivery + **idempotent/transactional processing** on the consumer side; broker-level EOS (Kafka transactions) only covers Kafka-in/Kafka-out topologies.
- **Head-of-line blocking**: within a partition, one poison message or slow item stalls everything behind it. Symptom: consumer lag climbs on one partition while others are fine. (This is exactly the mechanics behind AEMaaCS "blocked distribution queues" — single-partition topics → strict ordering → HOL blocking; see [content-distribution-journal](../cloud-service/content-distribution-journal.md).)
- **Rebalances**: consumer joins/leaves → partitions reassigned → processing pauses and in-flight work may duplicate. Long processing inside the poll loop triggers `max.poll.interval.ms` evictions → rebalance storms; move slow work out of the consumer thread or raise the interval deliberately.

## Retries

- Retry only **transient** failures (timeouts, 5xx, 429); retrying a deterministic failure (validation, 4xx) burns capacity and delays the DLQ verdict. Classify errors explicitly — the untyped `catch { retry }` is the root of most retry storms.
- **Exponential backoff + full jitter** (`sleep = rand(0, min(cap, base·2^attempt))`) — without jitter, synchronized clients re-converge into waves that re-kill the recovering dependency.
- **Retries require idempotency** downstream (see [[api-design]]); at-least-once delivery + non-idempotent consumer = duplicated side effects (double emails, double charges).
- Cap total retry time against the caller's timeout budget — retrying past the point where anyone still waits for the answer is pure load. Retry budgets (e.g. retries ≤ 10% of requests) stop retry amplification across service layers: 3 retries at each of 3 layers = up to 27× load on the bottom (see [[reliability-patterns]]).
- Queue-based retry: don't block the consumer sleeping — publish to a **delay/retry topic** (or use broker delay queues) with an attempt counter in headers, and keep the main partition flowing.

## Dead-letter queues (DLQs)

- A DLQ is where messages go after N failed attempts, so one poison message can't block the stream (the fix for HOL blocking) and so failures are *visible* rather than silently dropped.
- A DLQ without **alerting + a replay tool + an owner** is a write-only graveyard — the message is "handled" in the same sense a TODO is. Define: who gets paged, how messages are inspected (payload + failure reason + attempt metadata in headers), and how replay works (back to the main topic? direct reprocess? ordering with messages that came after?).
- Replay gotcha: a DLQ'd message re-enters *out of order* — consumers must tolerate that, or you need key-level quarantine (hold all later messages for that key too, expensive and rarely built).
- AEM instance: the distribution journal's error queue **cannot retry in place** — remove, fix, republish ([content-distribution-journal](../cloud-service/content-distribution-journal.md)).

## Reliable publishing: outbox and friends

The dual-write problem: "write DB + publish event" is two systems, no shared transaction — crash between them and state and stream disagree forever. Fixes:
- **Transactional outbox**: write the event into an `outbox` table in the same DB transaction as the state change; a relay (poller or CDC/Debezium) publishes from the table. At-least-once, so consumers dedupe (store processed message IDs, or make handling idempotent).
- **Listen-to-yourself / CDC-first**: derive events from the DB's change stream directly.
- The same atomic-commit trick appears in AEMaaCS: distribution offsets are stored in the **same commit** as the imported content → exactly-once across pod replacement ([content-distribution-journal](../cloud-service/content-distribution-journal.md)).

## Operational truths

- **Consumer lag is the health metric** (not queue "errors") — alert on lag growth rate and on max message age, per partition.
- Queues don't add capacity, they add **time**: a queue in front of an underprovisioned consumer just converts overload into unbounded latency. Bound the queue and shed load, or the backlog *is* your outage (see backpressure in [[reliability-patterns]]).
- Schema evolution is an API contract: additive changes only, tolerant readers, schema registry if the org is big enough to need one (same discipline as [[api-design]] versioning).
- Payload size: brokers hate big messages (Kafka default cap ~1 MB) — pass a reference to blob storage, not the blob. AEM does exactly this: binaries never travel in the distribution package, only references into the shared blob store.
