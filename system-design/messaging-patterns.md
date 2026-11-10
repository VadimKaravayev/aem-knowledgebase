# Messaging patterns — channels, routing, transformation

The enterprise integration patterns for systems that talk through messages: how messages are delivered, routed, reshaped and what happens to the ones that cannot be processed. Docs-based; pattern names follow Hohpe and Woolf's *Enterprise Integration Patterns*. The Camel snippets are illustrations and were not run. Related: [[async-systems]] (Kafka internals, retries, operating a dead-letter queue), [[distributed-transactions]] (saga, outbox, CDC), [[api-gateway-bff]] (synchronous integration at the edge), [[amqp]] (RabbitMQ's protocol).

## Why messaging

A message broker between two systems decouples them in three ways: **time** (the receiver need not be up when the sender sends), **rate** (a burst is buffered instead of overwhelming the receiver) and **knowledge** (the sender does not know who receives). The price is that everything becomes asynchronous: results arrive later, messages can arrive twice or out of order, and there is one more system to operate.

## Channels

### Point-to-point

One message is consumed by **exactly one receiver**. The channel is a queue. Used for **commands and work**: "charge this card", "resize this image" — something that must be done once.

### Publish-subscribe

One message is delivered to **every subscriber**, each getting its own copy. The channel is a topic. Used for **events**: "order placed" — a fact that any number of systems may care about, without the publisher knowing them.

| | Point-to-point | Publish-subscribe |
|---|---|---|
| Receivers per message | One | All subscribers |
| Message is | A command or task | An event (a fact) |
| Adding a consumer | Shares the existing work | Gets its own copy of everything; nobody else is affected |
| Sender knows the receiver | Implicitly — it expects the work done | No |
| RabbitMQ | One queue, one or more consumers | Fanout or topic exchange bound to one queue per subscriber |
| Kafka | One consumer group | Several consumer groups on one topic |

A subscription is **durable** if messages published while the subscriber is offline are kept for it. A non-durable subscriber simply misses them.

## Kafka and RabbitMQ

They are not two versions of the same thing: Kafka is a **distributed log**, RabbitMQ is a **message broker**.

| | Kafka | RabbitMQ |
|---|---|---|
| Core model | Append-only, partitioned log | Exchanges route messages into queues |
| After consumption | Message stays until retention expires; **can be re-read** | Message is deleted once acknowledged |
| Who tracks position | The consumer (an offset per partition) | The broker (per-message acknowledgement) |
| Delivery | Consumers pull batches | Broker pushes, limited by a prefetch count |
| Routing | By key to a partition; consumers filter | Rich: direct, topic (wildcards), fanout, headers exchanges |
| Ordering | Per partition, strict | Per queue, lost with several consumers or requeues |
| Per-message features | None built in — no priority, delay or per-message expiry | Priority, time-to-live, dead-letter exchanges, delayed delivery by plugin |
| Dead letters | A convention implemented in the consumer or framework | Built into the broker |
| Throughput | Very high; sequential disk writes and batching | Lower; ample for most task workloads |
| Parallelism limit | Number of partitions | Number of consumers |
| Protocol | Its own binary protocol | AMQP 0-9-1, also MQTT and STOMP |

- **Choose Kafka** when messages are events to keep: several independent consumers, replay, stream processing, high volume, feeding analytics, event sourcing.
- **Choose RabbitMQ** when messages are tasks to finish: work queues, flexible routing, per-message retry and expiry, request–reply, modest volume with simple operations.
- The frequent mistake is using Kafka as a task queue (one slow message blocks its partition, no per-message retry) or RabbitMQ as an event store (once consumed, the message is gone).
- The gap is narrowing from both sides — RabbitMQ has replicated quorum queues and an append-only stream type — but the default behaviours above are what a design should assume.

## Competing consumers

Several consumers read from the **same point-to-point channel**; each message goes to whichever is free. It is how message processing scales and survives the loss of a worker.

- **Throughput scales with the number of consumers**, and the queue depth is the natural autoscaling signal ([[autoscaling]]).
- **Ordering is lost**: message 2 may finish before message 1. Where order matters per entity, partition by key so that one consumer handles all messages for a key (Kafka partitions; a consistent-hash exchange in RabbitMQ).
- **A message is acknowledged only after it is processed.** A consumer that dies mid-work never acknowledges, and the broker redelivers to another — so delivery is **at-least-once** and every consumer must be **idempotent**.
- Tune how many unacknowledged messages one consumer may hold (prefetch). Too high and one consumer hoards the backlog while others idle; a slow consumer with a large prefetch also delays those messages.
- In Kafka the pattern is a consumer group, and the number of partitions caps the useful number of consumers.

## Poison messages and the dead letter channel

A **poison message** is one that can never be processed: malformed, missing a required field, referencing something that does not exist, or triggering a bug. Because failed messages are redelivered, a poison message is retried forever — burning capacity and, where order is preserved, **blocking everything behind it**.

The **dead letter channel** is where such messages are moved after a limited number of attempts, so the main flow continues and the failure is kept for inspection instead of being lost.

- **Distinguish transient from permanent failures.** A timeout deserves a retry with backoff. A validation error will fail identically every time — send it to the dead letter channel at once.
- **Count deliveries.** RabbitMQ dead-letters a message that is rejected without requeue, expires, overflows the queue or exceeds a delivery limit, to a configured dead-letter exchange. Cloud queues have a maximum receive count. **Kafka has no broker-side mechanism**: the consumer, or its framework, publishes the failed record to a separate topic and moves on.
- **Keep the evidence** with the message: original channel, error, attempt count, time of failure.
- A related pattern, the **invalid message channel**, is for messages the receiver cannot even interpret; many systems use one dead-letter queue for both.
- A dead letter channel nobody watches is a silent data-loss mechanism. It needs an alert, an owner and a way to replay — details in [[async-systems]].

## Routing patterns

### Content-based router

Inspects each message and sends it to one of several channels depending on its **content** — a field, a header, a type.

```java
from("jms:orders")
    .choice()
        .when(header("type").isEqualTo("digital")).to("jms:orders.digital")
        .when(jsonpath("$[?(@.total > 1000)]")).to("jms:orders.review")
        .otherwise().to("jms:orders.standard");
```

- The sender stays unaware of the receivers; the routing rules live in one place.
- That one place is also the cost: **the router must know every destination**, so each new receiver is a change to the router. When receivers should decide for themselves what they want, use publish-subscribe with filters on the subscriptions instead.
- Always have a default route. A message matching no rule and silently dropped is the classic defect.

### Splitter

Breaks **one message containing many items into one message per item**, so each can be routed and processed independently — an order into its order lines, a batch file into records.

```java
from("jms:orders")
    .split(jsonpath("$.items[*]"))
    .to("jms:order.items");
```

- Each part should carry what is needed to put the whole back together: a **correlation ID** shared by all parts, its **sequence number**, and the **total count**.
- Parts now succeed and fail independently. Decide what a half-processed order means before it happens.

### Aggregator

The reverse: collects **related messages and combines them into one**. It is the first stateful pattern here, and three decisions define it:

| Decision | Options |
|---|---|
| **Correlation** — which messages belong together | A shared correlation ID |
| **Completion** — when the group is done | All parts received, a fixed count, a timeout, an explicit "last" marker |
| **Aggregation** — how to combine | Concatenate, take the best (cheapest quote), sum, merge |

```java
from("jms:order.items.processed")
    .aggregate(header("orderId"), new MergeItemsStrategy())
        .completionSize(header("itemCount"))
        .completionTimeout(5000)
    .to("jms:orders.completed");
```

- **A timeout is mandatory.** One lost part otherwise leaves the group waiting forever and holding memory.
- **State must survive a restart**, or every in-flight group is lost; use a persistent store for the partial groups.
- Decide what happens to a part arriving **after** its group completed, and to **duplicates**.
- Splitter, parallel processing and aggregator together form **scatter-gather**: ask several suppliers for a quote, collect the answers that arrive in time, pick one.

## Transformation patterns

### Message translator

Converts a message from the format one system produces to the format another expects, so neither has to change. The classic case is XML to JSON:

```xml
<order id="456">
  <customer>123</customer>
  <items>
    <item sku="A-1" qty="2"/>
  </items>
  <postcode>00501</postcode>
</order>
```

```json
{
  "orderId": 456,
  "customerId": 123,
  "items": [ { "sku": "A-1", "quantity": 2 } ],
  "postcode": "00501"
}
```

Translation happens at several levels at once: the **representation** (XML → JSON, character encoding), the **data types** (text to number, date formats), and the **structure and names** (`id` attribute → `orderId`, `qty` → `quantity`).

Why a generic XML-to-JSON converter is not a translator:

- **The single-element array problem.** One `<item>` converts to an object, two to an array. Consumers written against one shape break on the other. The mapping must declare `items` to be an array always.
- **XML has no types without a schema.** Everything is text; a converter that guesses turns the postcode `00501` into the number `501`. Decide each field's type explicitly.
- **Attributes versus elements, namespaces, mixed content and element order** have no JSON equivalent, and each converter invents its own convention (`@id`, `#text`).
- **Empty element**: null, empty string or absent? Choose, and document it.

With many systems, translating between every pair is N × N mappings. A **canonical data model** — one shared format that each system translates to and from — makes it 2 × N, at the cost of agreeing on and governing that model.

### Envelope wrapper

Wraps the payload in an **envelope carrying the metadata the messaging infrastructure needs**, and unwraps it at the destination. The payload travels untouched.

```json
{
  "specversion": "1.0",
  "id": "b7c1e2a0-5d1f-4c1e-9a55-0d6f3f1c2e11",
  "source": "/services/order-service",
  "type": "com.example.order.created",
  "datacontenttype": "application/json",
  "correlationid": "4bf92f3577b34da6a3ce929d0e0e4736",
  "data": { "orderId": 456, "customerId": 123 }
}
```

- Typical envelope fields: message ID (for deduplication), type and schema version (for routing and parsing), source, time, correlation ID ([[logging]], [[distributed-tracing]]), and security or tenancy information.
- **Routers, filters and monitoring read only the envelope.** The payload can be in any format, compressed or encrypted, and intermediaries never need to understand it.
- A standard envelope across all services is what makes generic tooling possible. **CloudEvents** (the format above) is the common vendor-neutral one; a SOAP envelope is the older example.
- Keep business data out of the envelope and infrastructure data out of the payload. Where the transport has headers of its own (Kafka, AMQP), the envelope fields usually go there.

### Content enricher

Adds data that the message lacks and the receiver needs, by **looking it up from another source** — the event has a customer ID, the consumer needs the customer's country.

- The alternative is for the sender to include everything (a "fat" event). An enricher keeps senders simple and events small, and puts the lookup cost in the pipeline.
- **The lookup is a dependency of the flow**: it needs a timeout, a cache and a decision about what to do when the source is down ([[reliability-patterns]]).
- **Enrichment reads the data as it is now, not as it was when the event happened.** A replay next week enriches with next week's customer record. Where the historical value matters, the sender must put it in the event.
- Enriching message by message is an N+1 lookup; batch it or cache it.

### Claim check

Stores a **large payload in external storage and sends only a reference** — the claim check — through the messaging system. The receiver uses the reference to fetch the payload.

```json
{
  "type": "report.generated",
  "reportId": 9817,
  "payloadRef": "s3://reports-bucket/2f6c/report-9817-v1.pdf",
  "sizeBytes": 48211302,
  "sha256": "9f2c…"
}
```

- Why: brokers limit message size (around a megabyte by default in several), and large messages hurt throughput, memory and replication for every other message on the channel.
- A second use: keep **sensitive data** out of the broker and its intermediaries; only a holder of storage permissions can redeem the reference.
- Traps:
  - **Write the payload before publishing the message**, or a fast consumer redeems a reference to nothing.
  - **The payload must outlive every possible read**, including retries and dead-letter replays. Expire stored payloads later than the longest message retention; decide who deletes them.
  - **References should be immutable** (a versioned or content-addressed key). If the object is overwritten, a replayed message reads different content.
  - The consumer now depends on the storage service being available and on having access to it.

## Pattern summary

| Pattern | Problem it solves |
|---|---|
| Point-to-point channel | Exactly one receiver must do the work |
| Publish-subscribe channel | Many receivers must learn about an event |
| Competing consumers | Processing must scale and survive worker loss |
| Dead letter channel | A message cannot be processed and must not block or vanish |
| Content-based router | The destination depends on what the message contains |
| Splitter | A composite message must be handled item by item |
| Aggregator | Related messages must be combined into one |
| Message translator | Sender and receiver use different formats |
| Envelope wrapper | Infrastructure needs metadata the payload does not carry |
| Content enricher | The receiver needs data the sender does not have |
| Claim check | The payload is too large or too sensitive for the channel |

In Java these are provided by integration frameworks rather than written by hand — **Apache Camel** and **Spring Integration** both implement the catalogue under the same pattern names.

## What this means for AEM

- **Sling Jobs are competing consumers**: a job is processed by exactly one instance of those able to handle its topic, is persisted until acknowledged, and is retried on failure — so job consumers must be idempotent like any other at-least-once consumer.
- **Content distribution uses the claim check**: binaries are not sent inside the distribution package, only references into the shared blob store ([shared-datastore-binaryless-replication](../infrastructure-ops/shared-datastore-binaryless-replication.md), [content-distribution-journal](../cloud-service/content-distribution-journal.md)).
- **AEM events for external systems are publish-subscribe** through Adobe I/O Events, with the subscriber typically an App Builder action ([app-builder](../cloud-service/app-builder.md)).
