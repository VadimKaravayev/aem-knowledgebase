# Distributed transactions — 2PC, saga, outbox, CDC

Keeping data consistent when one business operation changes more than one database or service. Docs-based. Related: [[messaging-patterns]] (the channels these patterns run on), [[async-systems]] (retries, dead-letter queues), [[database-transactions-isolation]] (transactions inside one database), [[consistency-models]].

## The problem

Inside one database a transaction makes several changes atomic. Across two — an order database and a payment service, or a database and a message broker — there is no shared transaction. Any sequence of two writes can be interrupted between them, leaving one done and the other not.

There are two families of answer: make the writes atomic after all (**two-phase commit**), or accept that they are separate and make the system converge (**saga**, with **outbox** and **CDC** as the reliable plumbing).

## Two-phase commit (2PC)

A **coordinator** drives all participants through two phases:

1. **Prepare.** The coordinator asks every participant to prepare. Each does the work, writes it durably, holds its locks, and votes yes or no. A yes is a promise that it *can* commit, whatever happens next.
2. **Commit.** If every vote is yes, the coordinator records the decision and tells all to commit. If any vote is no, or a participant does not answer, it tells all to roll back.

What it gives: real atomicity across resources — all commit or none do.

What it costs:

- **Blocking.** A participant that has voted yes cannot decide by itself. If the coordinator fails at that moment, the participant stays *in doubt*, **holding its locks**, until the coordinator returns.
- **Availability multiplies down.** Every participant and the coordinator must be up for any transaction to complete.
- **Latency.** Two round trips and several forced disk writes, with locks held throughout.
- **Limited support.** Participants must implement the protocol (XA). Most message brokers used today, most NoSQL stores and every HTTP API do not.

Where it is still right: inside a distributed database, which runs it between its own shards with a replicated coordinator; and between a small number of XA-capable resources in one data centre. In Java the API is JTA with a transaction manager. Between independently owned services over a network, it is generally avoided.

## Saga

A saga replaces one distributed transaction with a **sequence of local transactions**. Each step commits in its own service, then triggers the next. If a step fails, the saga runs **compensating transactions** that undo the steps already completed, in reverse order.

```
Create order (PENDING) → Reserve stock → Charge payment → Confirm order
                                              ✗ fails
                         Release stock  ←─────┘
Cancel order          ←──┘
```

What that changes:

- **There is no isolation.** Between steps the intermediate state is real and visible: the order exists, the stock is reserved, the payment is not taken. Other requests can see and act on it.
- **Compensation is a business action, not a rollback.** A charge is undone by a refund, which is a second record, not the absence of the first. An email cannot be unsent.
- **Compensations must not fail permanently** — they are retried until they succeed — and, like every step, must be **idempotent**, because messages are delivered at least once.

Designing one:

- **Order the steps by reversibility.** Steps that can be compensated go first; then the **pivot** — the step after which the saga is committed to finishing; then steps that cannot fail for business reasons and are simply retried until done. Irreversible actions (shipping, sending a message to a customer) go after the pivot.
- **Make the in-between state explicit**: a `PENDING` status acts as a semantic lock, telling other operations that the record is mid-saga.
- **Every step needs a timeout**, and the saga a defined outcome for "no answer": retry, or compensate.

### Choreography

There is no coordinator. Each service **reacts to events** and publishes its own:

```
order-service ── OrderCreated ──▶ stock-service ── StockReserved ──▶ payment-service
      ▲                                                                    │
      └──────────────────────── PaymentCompleted ◀─────────────────────────┘
```

### Orchestration

A central **orchestrator** holds the saga's state and tells each participant what to do with **commands**, acting on their replies:

```
                    ┌──▶ stock-service     (ReserveStock  → reserved)
orchestrator ───────┼──▶ payment-service   (ChargePayment → failed)
                    └──▶ stock-service     (ReleaseStock)
```

| | Choreography | Orchestration |
|---|---|---|
| Control | Distributed; each service knows its own reactions | Central; one component knows the whole flow |
| Communication | Events | Commands and replies |
| Coupling | Services coupled to each other's events | Participants coupled to the orchestrator only |
| Seeing the flow | Implicit — spread over all services | Explicit — one definition, one state per saga |
| "Where is order 456 stuck?" | Requires tracing events across services | Read the saga's state |
| Adding a step | Touches several services | Change the orchestrator |
| Risks | Cyclic event dependencies; nobody owns the process | A "god" orchestrator absorbing business logic; one more service to run |
| Fits | Two to four steps, simple compensation | Longer flows, branching, timeouts, anything audited |

Orchestrators are usually built on a workflow engine that persists state and handles timers and retries (Temporal, Camunda, AWS Step Functions) rather than hand-written. The orchestrator should hold only the sequence and the compensation logic; the business rules stay in the participants.

## The dual-write problem

Every saga step does two things: change its database, and tell someone (publish an event or send a command). Those are two systems.

```java
orderRepository.save(order);              // 1. commit to the database
broker.publish(new OrderCreated(order));  // 2. publish — what if the process dies here?
```

Publish second and a crash loses the event: the order exists and nothing downstream ever hears. Publish first and a failed commit announces an order that does not exist. Reordering does not help; wrapping both in a database transaction does not include the broker.

## Outbox with polling

The **transactional outbox** turns the two writes into one. In the **same local transaction** as the business change, the service inserts the message into an `outbox` table. A separate **relay** reads that table and publishes.

```sql
BEGIN;
INSERT INTO orders (id, customer_id, status) VALUES (456, 123, 'PENDING');
INSERT INTO outbox (id, aggregate_id, type, payload)
VALUES ('b7c1e2a0-5d1f-4c1e-9a55-0d6f3f1c2e11', 456, 'OrderCreated', '{"orderId":456,"customerId":123}');
COMMIT;
```

Either both rows exist or neither does. The **polling publisher** is the simplest relay:

```sql
SELECT id, type, payload
FROM outbox
WHERE published_at IS NULL
ORDER BY created_at
LIMIT 100
FOR UPDATE SKIP LOCKED;
-- publish each row to the broker, then:
UPDATE outbox SET published_at = now() WHERE id = ANY(:published_ids);
```

- **Delivery is at-least-once.** A crash after publishing and before marking the row publishes it again. Consumers deduplicate on the message ID — commonly an **inbox** table of processed IDs written in the same transaction as the consumer's own change.
- `SKIP LOCKED` lets several relay instances run without publishing the same rows.
- Traps:
  - **Latency is the polling interval**, and frequent polling is constant load on the database.
  - **Insertion order is not commit order.** A transaction that took an earlier sequence number or timestamp can commit later; a poller that remembers "the highest ID I have seen" skips it. Select by unpublished status, not by a high-water mark.
  - **The table grows.** Delete or partition published rows.
  - Build the payload with a JSON serializer, never by string concatenation — a quote in a customer name otherwise produces a row that cannot be published.

## CDC — change data capture

**CDC** turns the changes committed to a database into a stream of events — one per inserted, updated or deleted row — that other systems consume.

| Method | How | Trade-off |
|---|---|---|
| **Log-based** | Reads the database's own transaction log (PostgreSQL write-ahead log, MySQL binary log) | Low overhead, sees every change including deletes, in commit order. Needs log access and a connector to operate |
| **Trigger-based** | Triggers copy each change into a change table | Works anywhere; adds work to every write |
| **Query-based** | Polls for rows with `updated_at` later than the last run | Trivial to set up; **misses deletes and intermediate changes**, depends on a reliable timestamp |

Log-based is what "CDC" usually means. **Debezium** (connectors running on Kafka Connect) is the common open-source implementation; cloud databases offer their own change streams.

Uses:

- **Outbox without polling**: the connector tails the log for inserts into the outbox table and publishes them — no polling load, low latency, and commit order preserved. This is the preferred relay once CDC infrastructure exists.
- Keeping **derived stores** in step with the source of truth: search index, cache, data warehouse, read models.
- **Migrations**: streaming changes from a legacy database into its replacement while both run.

Traps:

- **Raw table changes are not a contract.** Consumers reading the `orders` table's change stream are coupled to its columns; every schema change becomes a breaking change for unknown readers. Publishing through an outbox table gives a deliberate, versioned event instead.
- **A stalled connector can take the database down.** With PostgreSQL the replication slot makes the server keep all log segments the connector has not read; if the connector is stopped, the disk fills. Monitor slot lag.
- **Starting up** requires a consistent snapshot of existing data followed by a clean hand-over to streaming.
- Delivery is at-least-once and ordered per key, not globally. Deletes arrive as tombstones that consumers must handle.

## Choosing

| Situation | Use |
|---|---|
| Several tables in one database | A local transaction — no pattern needed |
| A few XA-capable resources, short transactions, same data centre | 2PC, knowingly |
| A business process across services | Saga — choreography when short, orchestration when long or audited |
| A service must change its database and publish a message | Outbox, relayed by polling or CDC |
| Other systems need to follow a database's data | CDC |

The first question is always whether the boundary can be moved so that the data that must change together lives in one service. A saga that exists only because one aggregate was split across two services is a design problem, not an integration problem.
