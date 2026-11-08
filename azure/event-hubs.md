# Azure Event Hubs: partitioned event log as a service

Azure Event Hubs is Microsoft's managed **event streaming** service: an append-only, partitioned log you write events into and read back by position, retained for a fixed time. It is the Kafka-shaped member of the Azure messaging family and is routinely confused with the other two:

| Service | Model | Pick it when |
|---|---|---|
| **Event Hubs** | Partitioned log; readers track their own position; events expire by retention, never by being read | Telemetry, clickstreams, change feeds, anything with high volume where several independent readers each need the whole stream and replay matters |
| Service Bus | Queues and topics; per-message lock, complete/abandon, dead-letter queue, sessions, duplicate detection, transactions | Commands and work items that must be processed exactly once by one of N workers |
| Event Grid | Push router for discrete events to webhooks, functions, MQTT clients; no retention beyond retry | "Something happened, call these endpoints" with no replay requirement |

This doc is docs-based (Azure Event Hubs documentation and quotas pages, Java SDK v5 behaviour). Quotas move; where a number matters for a design, re-check the current "Event Hubs quotas and limits" page before committing. The wire protocol underneath is AMQP 1.0, covered in `amqp`; the design question of log versus queue is in `async-systems`.

## The tape-deck picture

An event hub is a rack of numbered tape decks (partitions). Every writer appends to the end of one tape; which tape is decided by the partition key, or round-robin if there is none. A reader picks a tape and a position and plays forward, keeping its own bookmark. Tapes are overwritten from the start after the retention period, whether anyone played them or not. Nothing on the rack knows or cares who has listened to what.

What follows from the picture:

- **Ordering exists only within one tape.** Two events with different partition keys have no defined order relative to each other. Same key, same tape, strict order.
- **Reading is non-destructive and the service keeps no bookmark for you** (one exception: Kafka-protocol consumer groups, below). Lose your checkpoint and you replay from wherever you choose to start.
- **Parallelism is the number of tapes.** One reader per partition per consumer group is the design; more readers than partitions sit idle, and a partition count is fixed at creation on the Basic and Standard tiers.
- **There is no per-event delete, lock, dead-letter or TTL.** Poison events are the reader's problem: skip, checkpoint past, and write the bad one somewhere else.

## Object hierarchy

```
Namespace          <name>.servicebus.windows.net   (yes, the Service Bus domain; one namespace is one SKU, one region, one billing unit)
└── Event Hub      the "topic"; retention, partition count, capture and cleanup policy live here
    ├── Partitions 0..N-1
    │     each: ordered sequence of events, each with sequence number (int64, gap-free per partition),
    │           offset (opaque string position, not comparable across partitions), enqueued time (UTC)
    └── Consumer groups   "$Default" always exists; a named view of the whole hub, one position per partition per group
          └── Readers     at most 5 concurrent per partition per group; 1 is the design
```

Partition counts: Basic and Standard 1 to 32 per hub (fixed after creation; more by support ticket), Premium up to 100, Dedicated up to 1024, and the Premium and Dedicated tiers can **increase** (never decrease) the count on a live hub. Increasing re-shards the key space: events with a given key start landing on a different partition from that moment, so key-based ordering has a seam at the resize.

Each partition also has its own throughput ceiling (1 MB/s ingress, 2 MB/s egress on Standard), so **buying more throughput units than you have partitions buys nothing**; size partitions for peak throughput and for the reader parallelism you want, not for today's rate.

## Protocols and endpoints

| Endpoint | Port | Use |
|---|---|---|
| AMQP 1.0 | 5671 (TLS) | Native SDKs (`azure-messaging-eventhubs` for Java/.NET/Python/JS, built on Qpid Proton or the Azure AMQP core). Send and receive. |
| AMQP over WebSockets | 443 | Same SDKs with the transport switched (`AmqpTransportType.AMQP_WEB_SOCKETS`); the escape hatch for networks that only allow 443. |
| HTTPS REST | 443 | **Send only**: `POST https://<ns>.servicebus.windows.net/<hub>/messages` (or `/partitions/<id>/messages`) with a SAS token in `Authorization`. There is no HTTP receive. |
| Kafka | 9093 | Any Kafka 1.0+ client, `SASL_SSL`, mechanism `PLAIN` with user `$ConnectionString` and the connection string as password, or `OAUTHBEARER` with an Entra token. **Not available on the Basic tier.** No code change beyond `bootstrap.servers` and the SASL block; no ZooKeeper, no broker to manage. |

Kafka mapping: event hub = topic, consumer group = Kafka consumer group (and **these offsets are committed server-side**, the one place Event Hubs keeps a bookmark for you), namespace = cluster. Feature gaps exist and shift between releases (transactions, some compression codecs, Kafka Streams, idempotent producer semantics have each been unsupported or preview at various points); read the "Event Hubs for Apache Kafka: feature support" list for the version you are on rather than assuming parity. Topic auto-creation can be turned on for the namespace; by default a Kafka producer writing to a non-existent topic gets `UNKNOWN_TOPIC_OR_PARTITION`.

Authentication is one of two: **shared access signatures** (namespace- or hub-scoped policies with `Send`, `Listen`, `Manage` rights; the connection string is `Endpoint=sb://<ns>.servicebus.windows.net/;SharedAccessKeyName=<policy>;SharedAccessKey=<key>;EntityPath=<hub>`) or **Microsoft Entra ID RBAC** with the roles Azure Event Hubs Data Owner / Data Sender / Data Receiver, usually via a managed identity. Local (SAS) auth can be disabled on the namespace, which the security baseline asks for. Network isolation: IP firewall, VNet service endpoints, or Private Link; the Kafka port follows the same rules.

## Writing

An event is up to **1 MB** (256 KB on Basic) including system properties, and a batch is capped at the same 1 MB. The producer decides which partition in one of three ways:

- **Partition key** (`setPartitionKey`): the service hashes it to a partition. This is the right default: it gives per-key ordering and survives the service moving partitions between nodes.
- **No key**: round-robin across partitions. Maximum spread, no ordering.
- **Explicit partition id**: pinned. Discouraged, because an unavailable partition then means a failed send instead of a redirected one, and because it makes the partition count part of your code.

Events carry a body (bytes), **application properties** (your string-keyed map, where a Kafka client's headers end up) and **system properties** set by the service (`x-opt-sequence-number`, `x-opt-offset`, `x-opt-enqueued-time`, `x-opt-partition-key`). A client-side `message-id` is just a property; **the service does not deduplicate**, so a retried batch after a timeout is a duplicate batch, and idempotency belongs to the consumer.

Throttling is the main production failure. Standard-tier capacity is in **throughput units (TU)**: 1 TU = 1 MB/s or 1,000 events/s ingress, 2 MB/s or 4,096 events/s egress, 1 to 40 per namespace self-serve. Exceed it and sends fail with `ServerBusyException` (AMQP error `com.microsoft:server-busy`; Kafka producers see throttling delays). **Auto-inflate** raises TUs up to a ceiling you set, never lowers them, and reacts over minutes, so a burst still throttles first. The SDK retries server-busy with backoff; your code should treat a persistent one as "add TUs or partitions", not as a bug to catch.

## Reading and checkpointing

A reader opens a link to one partition in one consumer group and names a starting position: an offset, a sequence number, an enqueued time, `@latest` (only new events) or the earliest retained (`-1`). From there it receives in batches (`maxBatchSize`, `maxWaitTime`) forever. Everything about position is the client's responsibility, and the SDK packages it as **`EventProcessorClient`**:

- **Checkpoint store**: Azure Blob Storage by default (`azure-messaging-eventhubs-checkpointstore-blob`). One container holds blobs named `<namespace>/<hub>/<consumer-group>/checkpoint/<partition>` whose **metadata** carries the offset and sequence number, and `.../ownership/<partition>` blobs whose metadata names the owning processor with an ETag used for optimistic claims. The data is all in metadata; the blobs are empty.
- **Load balancing**: every processor instance using the same store and consumer group periodically (about every 10 s) lists ownership blobs, claims unowned or expired partitions (ownership expires after about 2 min without renewal), and steals from instances that own more than their fair share until each has `partitions / instances`. Adding an instance causes a rebalance; a partition can be briefly read by two instances during the hand-over, which is one of the reasons the whole system is **at-least-once**.
- **Owner level (epoch)**: a reader with a higher owner level evicts any lower one on the same partition and consumer group with `ReceiverDisconnectedException`. The processor uses it; two processors on the same consumer group with different checkpoint containers fight each other through it, which looks like consumers randomly dying.
- **When to checkpoint**: after processing, not on receipt, and not per event (each checkpoint is a blob write). Checkpoint every N events or seconds; on a crash you replay back to the last checkpoint, so N bounds the duplicate window.

Reading does not consume anything, so the same hub feeds analytics, archival and an application through three consumer groups without interference. Consumer groups per hub: Basic 1 (`$Default` only), Standard 20, Premium 100, Dedicated 1000.

## Retention, capture, compaction

- **Retention** is per hub and time-based: Basic 1 day, Standard 1 to 7 days, Premium and Dedicated up to 90 days. Stored size is not a limit you set; you pay for it on Premium and Dedicated and it is included on Standard. Expired events vanish whether or not anyone read them, and a reader whose checkpoint is older than retention starts from the earliest surviving event with no error.
- **Capture** (Standard and above) writes every partition's events to Blob Storage or Data Lake Storage Gen2 as **Avro files**, cut by a time window (1 to 15 min) and a size window (10 to 500 MB), whichever comes first, in a path you template (`{Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}`). It does not consume egress throughput and needs no reader of your own; the Avro wraps the body as bytes plus the system properties, so a downstream job has to decode your payload format itself. Empty windows produce empty files unless "skip empty archives" is on.
- **Cleanup policy**: `Delete` (plain time retention) or `Compact` (log compaction: keep the latest event per partition key, Kafka-style; Standard and above). Compacted hubs are how you get a "current state per key" feed; they still need a key on every event.
- **Schema Registry** lives on the namespace (Standard and above): Avro and JSON schemas with compatibility rules, referenced from events by schema id via the SDK serializers. It is opt-in; the service never validates payloads itself.

## Tiers and the quotas that bite

| | Basic | Standard | Premium | Dedicated |
|---|---|---|---|---|
| Capacity unit | TU | TU (1 to 40, auto-inflate) | Processing unit (1, 2, 4, 8, 16); roughly 5 to 10 MB/s ingress each | Capacity unit; single-tenant cluster, self-serve |
| Partitions per hub | 32 | 32 (fixed) | 100 (growable) | 1024 (growable) |
| Consumer groups per hub | 1 | 20 | 100 | 1000 |
| Event hubs per namespace | 10 | 10 | 100 per PU | 1000 per CU |
| Retention | 1 day | 1 to 7 days | to 90 days | to 90 days |
| Max event / batch size | 256 KB | 1 MB | 1 MB | 1 MB |
| Brokered (AMQP) connections per namespace | 100 | 5,000 | 10,000 per PU | 100,000 per CU |
| Kafka endpoint | no | yes | yes | yes |
| Capture | no | yes (extra charge) | included | included |
| Schema Registry | no | yes | yes | yes |
| Billing | TU-hour + per million events | TU-hour + per million events | PU-hour, no per-event charge | CU-hour |

Things people find out late:

- **Ten event hubs per Standard namespace.** A "hub per tenant" or "hub per entity type" design runs out at ten; the pattern is one hub with a discriminator property, or more namespaces, or Premium.
- **Basic to Standard is an in-place SKU change; Standard to Premium is a new namespace** (different hostname unless you plan the name) and a migration of clients and checkpoints.
- **Brokered connections count every AMQP link holder**, including each `EventProcessorClient` partition reader and each Kafka consumer; a fleet of microservices with per-instance clients hits 5,000 sooner than expected and gets `QuotaExceededException`.
- **Geo-disaster recovery on Standard is metadata only**: the alias fails over hubs, consumer groups and policies to the paired namespace, **not the events**, and failover is one-way (re-pairing is manual). Replicating data needs the Premium/Dedicated geo-replication feature or your own dual-write.
- **A partition's events are served by one node**; a hot key (one tenant producing half the traffic) throttles that partition at its ceiling while the namespace still has headroom. Spread with a composite key or accept the ceiling.

## Observing it

- Namespace **metrics**: `IncomingMessages`, `OutgoingMessages`, `IncomingBytes`, `OutgoingBytes`, `ThrottledRequests` (the one to alert on), `QuotaExceededErrors`, `ServerErrors`, `ActiveConnections`, `CaptureBacklog`, `Size`. Dimensions include entity name, so per-hub charts are possible. Consumer lag is **not** a service metric because the service does not know your checkpoints; compute it as last-enqueued sequence number (available on the partition properties call, or via `trackLastEnqueuedEventProperties` on the consumer) minus checkpointed sequence number.
- **Diagnostic settings** ship operational, auto-scale, Kafka coordinator, Kafka user-error, customer-managed-key and runtime audit logs to Log Analytics; "Kafka user error logs" is where a misconfigured `sasl.jaas.config` or missing topic shows up, since the Kafka client itself only reports a generic auth or metadata failure.
- The portal **Data Explorer** on a hub can send a test event and peek recent events per partition without a client.
- A local **Event Hubs emulator** (container image) exists for development; it speaks AMQP and Kafka with a fixed connection string and has no Capture, Schema Registry or Entra auth.

## What this means for AEM

- **AEM is only ever a producer or an HTTP-fed sink, never a well-placed consumer.** Producing from AEM is a short Sling job or event handler calling `EventHubProducerClient` with the content path as partition key, one long-lived client per instance (it is a cached AMQP connection, not a per-call object) and a bounded send timeout. Consuming inside AEM would make every publish pod an `EventProcessorClient` instance fighting over partitions through blob leases, with JCR writes as the side effect; put an Azure Function or container app on the hub and have it call AEM over HTTPS (Sling servlet, Assets API) or Adobe I/O instead.
- **On AEM as a Cloud Service the transport is decided by egress.** Default egress is ports 80 and 443 only, so use the SDK with `AMQP_WEB_SOCKETS` or the REST send endpoint; AMQP 5671 and Kafka 9093 need flexible port egress through the Adobe-provided proxy or a VPN, and the SDK must be told about that proxy. On 6.5 or AMS it is a firewall rule either way. Authentication from AEM is a SAS policy scoped to the one hub with `Send` only, stored as a Cloud Manager secret environment variable or in an OSGi configuration via `$[secret:...]`; managed identity is not available to an AEM runtime.
- **Adobe-side events rarely belong in Event Hubs directly.** Adobe I/O Events (AEM page or asset events, Journaling) deliver by webhook or a pull API over HTTPS; the normal bridge is a small Azure Function that receives the webhook and forwards to the hub, which also gives you the place to add the partition key and the idempotency key Event Hubs itself will not enforce.

## Cheat sheet

| | |
|---|---|
| Hostname | `<namespace>.servicebus.windows.net`; AMQP 5671, WebSockets/REST 443, Kafka 9093 |
| Unit of order | one partition; partition key → hash → partition |
| Position | sequence number (per-partition int64), offset (opaque string), enqueued time; `@latest` / `-1` to start |
| Who tracks position | the reader (blob checkpoint store); Kafka consumer groups are the exception |
| Delivery | at-least-once, no dedup, no lock, no DLQ, no per-event TTL |
| Capacity | Standard TU = 1 MB/s in, 2 MB/s out; per-partition ceiling the same, so partitions ≥ TUs |
| Fixed at creation (Standard) | partition count, tier (to Premium), region |
| Throttled | `ServerBusyException` / `com.microsoft:server-busy`; metric `ThrottledRequests` |
| Evicted reader | `ReceiverDisconnectedException` = higher owner level took the partition |
| Archive | Capture → Avro in Blob/ADLS by time (1 to 15 min) and size (10 to 500 MB) windows |
| Neighbours | Service Bus for work queues, Event Grid for push, Kafka if you want to own the cluster |
