# AMQP: anatomy of the protocol

AMQP (Advanced Message Queuing Protocol) is a binary, connection-oriented wire protocol for passing messages between an application and a broker (or, in 1.0, between any two peers). Two things that go by the name share nothing but the first four bytes of the handshake, and most confusion about AMQP comes from mixing them up:

| Name | Who speaks it | What it is |
|---|---|---|
| **AMQP 0-9-1** | RabbitMQ (native), plus everything that targets RabbitMQ (Spring AMQP, Pika, amqplib, Bunny, ...) | Broker model baked into the protocol: exchanges, queues, bindings, channels, acks. Published 2008. The one people usually mean by "AMQP". |
| **AMQP 1.0** | Azure Service Bus / Event Hubs (see `event-hubs`), Apache ActiveMQ Classic and Artemis, Apache Qpid, Solace, IBM MQ; RabbitMQ 4.x natively (3.x via plugin) | OASIS 2012, ISO/IEC 19464:2014. A peer-to-peer transfer protocol with a self-describing type system and credit-based flow control. **No exchanges or queues in the spec**; what you address is a broker-defined "node". |

Both default to TCP 5672 plain and 5671 TLS (URI schemes `amqp://` and `amqps://`). A 0-9-1 client cannot talk to a 1.0-only broker and vice versa; the handshake below is where that fails. Companion docs: [tcp.md](tcp.md) (everything here rides a single TCP byte stream), [tls.md](tls.md), and `async-systems` for the queue-vs-log design question that decides whether AMQP is the right tool at all.

Most of this doc is 0-9-1, because that is the one with a wire format worth memorising; 1.0 gets its own section. Facts are spec-based (AMQP 0-9-1 spec with RabbitMQ's errata, AMQP 1.0 core spec) with RabbitMQ defaults called out where the spec leaves a choice.

## The switchboard picture

A 0-9-1 connection is a phone line to the broker's switchboard. On that one line you open many **channels**, each an independent conversation with its own operator; the line is expensive (TCP + TLS + authentication), channels are cheap (two frames). You never hand a message to a queue. You hand it to an **exchange** with a **routing key** written on it; the exchange looks up its **bindings** and copies the message into zero or more **queues**; a **consumer** subscribed to a queue gets it pushed down a channel and must **acknowledge** it before the broker forgets it. Every noun in that sentence is a protocol object with declare/bind/delete methods.

The consequences that matter in practice:

- **An exchange with no matching binding drops the message silently** unless you asked for `mandatory` (then you get a `Basic.Return`). "Published fine, nothing arrived" is almost always a binding or routing-key problem, not a network one.
- **The queue is the only storage.** A message acked by a consumer is gone; there is no replay and no offset. If you need "read it again next week", you want a log (Kafka, RabbitMQ Streams), not a queue.
- **A channel error kills the channel, not the connection**, and every unacked message on that channel is requeued and redelivered with `redelivered=true`. Libraries that reuse one channel for everything turn a single `404 NOT_FOUND` on a queue declare into a stalled consumer.

## The handshake

The client speaks first with an 8-byte **protocol header**, not a frame:

```
41 4D 51 50  00  00  09  01      "AMQP" + protocol-id 0 + major 0 + minor 9 + revision 1
```

Other headers you will meet: `AMQP 01 01 08 00` (0-8, legacy), `AMQP 01 01 00 09` (0-9), `AMQP 00 01 00 00` (1.0 plain), `AMQP 03 01 00 00` (1.0 SASL layer), `AMQP 02 01 00 00` (1.0 TLS layer). A broker that does not speak the offered version **replies with the header it does speak and closes the socket**. That is the entire version negotiation, and it is why a mis-targeted client fails with "connection closed by peer" or "unexpected protocol header" before any authentication happens. You can test a broker with nothing but `netcat`:

```
$ printf 'AMQP\x00\x00\x09\x01' | nc -w 2 broker.example.net 5672 | xxd | head -5
00000000: 0100 0000 0001 f300 0a00 0a00 0901 d000  ................
00000010: 0000 ...  0c63 6170 6162 696c 6974 6965  .... capabilitie
00000020: 73 ...  0770 726f 6475 6374 5300 0000 08  s ... productS....
00000030: 5261 6262 6974 4d51                      RabbitMQ
```

Read it: frame type `01` (method), channel `0000`, size, class `000a` (connection), method `000a` (start), version `00 09`, then a server-properties table in which the product name, version and capabilities are sent in clear text **before any credentials**. A 1.0-only broker answers the same probe with `AMQP\x00\x01\x00\x00` and a FIN.

After the header, everything is frames on channel 0:

```
client → AMQP header
server → Connection.Start      version, server-properties, mechanisms ("PLAIN AMQPLAIN", plus EXTERNAL for client certs), locales
client → Connection.Start-Ok   client-properties (library name/version; RabbitMQ shows these in the management UI), mechanism, SASL response (PLAIN = \0user\0password), locale
        [Connection.Secure / Secure-Ok]   only for multi-step SASL mechanisms
server → Connection.Tune       channel-max, frame-max, heartbeat   (RabbitMQ proposes 2047, 131072, 60)
client → Connection.Tune-Ok    the client may lower any of the three, never raise them
client → Connection.Open       virtual-host ("/" by default; in a URI the vhost is the path, so "/" becomes "%2F")
server → Connection.Open-Ok
client → Channel.Open  (channel n ≥ 1) / server → Channel.Open-Ok
```

Authentication failures are reported **inside** this exchange: a wrong password on RabbitMQ produces `Connection.Close` with reply code `403 ACCESS_REFUSED` (older versions simply closed the socket), and a vhost the user cannot see produces the same `403` on `Connection.Open`, so "403 on open" means permissions, not credentials.

## Frame anatomy

Every byte after the protocol header belongs to a frame. The frame header is 7 bytes, the trailer 1, and both are the same for all frame types:

```
 0        1                 3                         7                        7+size
+--------+--------+--------+--------+--------+--------+--------+ ... +--------+
|  type  |     channel     |              size (payload bytes)       | payload |  0xCE  |
+--------+--------+--------+--------+--------+--------+--------+ ... +--------+
```

- **Type (1 byte).** `1` METHOD, `2` HEADER (content header), `3` BODY, `8` HEARTBEAT. (`4` was OOB in 0-9 and is gone.)
- **Channel (2 bytes).** `0` for connection-level methods and heartbeats; `1`..channel-max otherwise. Frames of different channels interleave freely on the socket; frames of one content message on one channel must not.
- **Size (4 bytes).** Payload length only, excluding the 7-byte header and the end byte. Must not exceed the negotiated frame-max; a larger frame is a connection error `501 FRAME_ERROR`.
- **Frame-end (1 byte).** Always `0xCE`. It is a sanity check, not a delimiter: parsers read `size` bytes and then *verify* the `CE`. Anything else is `501 FRAME_ERROR` and the connection dies.

**Method frame payload**: `class-id (2) | method-id (2) | arguments`. The arguments are a fixed sequence per method, encoded with the 0-9-1 primitive types: `octet`, `short`, `long`, `longlong`, `bit` (packed 8 to a byte, consecutive bits share one byte), `shortstr` (1-byte length, max 255 bytes, so **routing keys, queue and exchange names are capped at 255 bytes**), `longstr` (4-byte length), `timestamp` (64-bit seconds), `table` (longstr-sized list of `shortstr key, type-tag, value`; the `headers` property and all `x-` queue arguments are tables).

Class and method ids you will see in captures:

| Class | id | Methods (id) |
|---|---|---|
| connection | 10 | start 10/11, secure 20/21, tune 30/31, open 40/41, close 50/51 |
| channel | 20 | open 10/11, flow 20/21, close 40/41 |
| exchange | 40 | declare 10/11, delete 20/21, bind 30/31, unbind 40/51 |
| queue | 50 | declare 10/11, bind 20/21, purge 30/31, delete 40/41, unbind 50/51 |
| basic | 60 | qos 10/11, consume 20/21, cancel 30/31, **publish 40**, return 50, **deliver 60**, get 70 (get-ok 71, get-empty 72), **ack 80**, reject 90, recover 110/111, **nack 120** |
| confirm | 85 | select 10/11 (RabbitMQ extension) |
| tx | 90 | select 10/11, commit 20/21, rollback 30/31 |

Odd second ids are the `-Ok` replies; methods with no reply (publish, deliver, ack, nack, reject) are "asynchronous" in spec terms and are exactly the ones whose failures you only learn about later via `Channel.Close`.

## How a message crosses the wire

Four methods carry content: `Basic.Publish` (client to broker), `Basic.Deliver`, `Basic.Return` and `Basic.Get-Ok` (broker to client). Each is **always three or more frames in a fixed order on one channel**:

```
1. METHOD  frame   class 60, method 40 (publish): reserved short, exchange shortstr, routing-key shortstr, bits [mandatory, immediate]
2. HEADER  frame   class-id 60, weight 0, body-size (64-bit, total bytes of all body frames), property-flags (16-bit), properties
3. BODY    frame(s) raw bytes, split at (frame-max − 8) per frame; a 1 MB message at the default frame-max is 8 body frames
```

A header frame with body-size 0 is followed by no body frames; that is a legal empty message. Because the frames are a unit, a parser that sees a method frame on the same channel before the announced body bytes have arrived raises `505 UNEXPECTED_FRAME` and closes the connection.

**The property-flags word and the 14 Basic properties** (present only if its flag bit is set; the top bit first):

| Property | Type | What it is for / gotcha |
|---|---|---|
| `content-type` | shortstr | MIME type. Advisory; the broker never looks. |
| `content-encoding` | shortstr | e.g. `gzip`. Advisory. |
| `headers` | table | Arbitrary key/values; the **headers exchange** matches on these. `x-death` (dead-lettering history) and `x-delay` live here. |
| `delivery-mode` | octet | `1` transient, `2` persistent. **Persistent only survives a broker restart if the queue is also durable**; one without the other buys nothing. |
| `priority` | octet | 0-9; honoured only by queues declared with `x-max-priority`, ignored otherwise. |
| `correlation-id` | shortstr | RPC: echo the request's id in the reply. |
| `reply-to` | shortstr | RPC: queue name to answer on. |
| `expiration` | shortstr | **Per-message TTL in milliseconds as a string** (`"60000"`), not a timestamp. Expired messages at the head of a queue are dropped; expired ones behind a live head stay until they reach the head (classic queues). |
| `message-id` | shortstr | App-assigned; the broker does not dedupe on it. |
| `timestamp` | timestamp | Seconds, not milliseconds. |
| `type` | shortstr | App-defined. |
| `user-id` | shortstr | **The one property the broker validates**: RabbitMQ rejects a publish whose `user-id` differs from the authenticated user (`406 PRECONDITION_FAILED`). Use it when consumers must trust the sender. |
| `app-id` | shortstr | App-defined. |
| `cluster-id` | shortstr | Reserved, unused. |

Size limits are the broker's, not the protocol's: a 0-9-1 body size is a 64-bit field, but RabbitMQ enforces `max_message_size` (default 128 MiB in 3.8 to 3.13, **16 MiB from 4.0**, hard ceiling 512 MiB) and closes the channel with `406 PRECONDITION_FAILED` on anything bigger.

## Routing: exchanges and bindings

An exchange is a routing table with a type; a binding is a row `(exchange, queue, binding-key, arguments)`.

| Type | Matches when | Use |
|---|---|---|
| `direct` | `routing-key == binding-key` | Point-to-point, work queues. The nameless **default exchange `""`** is a direct exchange to which every queue is auto-bound under its own name, which is how "publish to a queue" works in tutorials. |
| `fanout` | always (key ignored) | Broadcast. |
| `topic` | key is dot-separated words; binding uses `*` (exactly one word) and `#` (zero or more) | `order.*.created`, `logs.#`. **`*` is not a wildcard inside a word**: `order.*` does not match `order.us.created`. |
| `headers` | `headers` table matches binding args; `x-match: all|any` | Rare; slower; use when the key would need to be multi-dimensional. |

Exchanges can be bound to exchanges (RabbitMQ `Exchange.Bind`), which is how one topic exchange fans into per-tenant direct exchanges. A message is copied once per matching queue, and a queue bound twice to the same exchange still gets one copy.

**Declare is idempotent only if the arguments match.** `Queue.Declare` on an existing queue with a different `durable`, `exclusive`, `auto-delete` or any `x-` argument returns `406 PRECONDITION_FAILED` and closes the channel. Changing a queue's type or dead-letter exchange means delete-and-recreate (or a policy, which is applied broker-side and does not count as an argument). `passive=true` turns a declare into an existence check that fails with `404 NOT_FOUND` instead of creating.

Queue properties in the declare: `durable` (survives broker restart; still loses transient messages), `exclusive` (bound to this connection, deleted when it closes; used for reply queues), `auto-delete` (deleted when the last consumer cancels; **never deleted if it never had a consumer**). Queue arguments that shape behaviour: `x-message-ttl`, `x-expires` (queue TTL when unused), `x-max-length` / `x-max-length-bytes` (default drop-head overflow, `x-overflow: reject-publish` to push back), `x-dead-letter-exchange` / `x-dead-letter-routing-key`, `x-max-priority`, `x-queue-type` (`classic`, `quorum` for Raft-replicated, `stream` for a log with offsets accessed over 0-9-1 with `x-stream-offset` on consume).

## Delivery, acknowledgement and flow control

Two ways to receive:

- **`Basic.Consume`** registers a consumer (tag) on a queue; the broker *pushes* `Basic.Deliver` frames as messages arrive. This is the normal path.
- **`Basic.Get`** pulls one message, synchronously, with a round trip per message. It exists for scripts and debugging; a service polling with Get is the classic mis-use (the RabbitMQ docs call it "highly inefficient" next to a consume).

Each delivery carries a **delivery-tag**, a 64-bit counter **scoped to the channel**, starting at 1. Acks reference it: `Basic.Ack(tag, multiple)` (multiple=true acks every tag up to and including this one), `Basic.Nack(tag, multiple, requeue)` (RabbitMQ extension), `Basic.Reject(tag, requeue)` (one message, no multiple). Acking a tag on a different channel than the delivery came on is `406 PRECONDITION_FAILED: unknown delivery tag` and closes the channel; so is acking twice. `requeue=false` on nack/reject either drops the message or dead-letters it if the queue has a DLX; `requeue=true` returns it to its original position (classic) or near the head (quorum) and it comes back with `redelivered=true`, which is also what happens to every unacked message when the channel or connection dies. Quorum queues count redeliveries in `x-delivery-count` and can `delivery-limit` them to a DLX; classic queues will redeliver a poison message forever.

`no-ack=true` on consume (library names: `autoAck`) means the broker forgets the message as it writes it to the socket. Combined with a slow consumer this is the unbounded-memory failure mode: there is nothing to throttle the push except TCP backpressure.

**Prefetch (`Basic.Qos`)** is the only throttle. `prefetch-count` caps unacked deliveries in flight; `0` means unlimited and is the default. The `global` bit is where the spec and RabbitMQ diverge: in RabbitMQ `global=false` (the default in most libraries) applies the limit **per consumer**, `global=true` **per channel shared across its consumers**; the spec said the opposite scope (per connection). Set prefetch on every consuming channel; with the default of unlimited, one consumer on a channel can drain a queue of a million messages into its heap while its siblings starve, and nothing is acked until it finishes.

Competing consumers on one queue get messages **round-robin in order of readiness**, bounded by each one's prefetch, so per-queue FIFO is only guaranteed with a single consumer; with several, two messages can be processed in either order and a redelivery jumps the queue. Ordering is a property of one queue and one consumer, never of an exchange.

**Broker-side flow control** is `Channel.Flow(active=false)`, which asks the client to stop publishing; RabbitMQ no longer sends it and instead **stops reading from the socket** of a publishing connection whose queues cannot keep up (the connection shows `flow` in the management UI); when a memory or disk alarm fires it additionally sends the extension method `Connection.Blocked` and the connection shows `blocked`. From the publisher's side that looks like a `Basic.Publish` that blocks inside the TCP send buffer, with no error.

## Publisher confirms and transactions

`Basic.Publish` has no reply. Without more, "the publish method returned" means the bytes left the client; a broker that crashes or a routing error you did not ask to hear about loses the message with no signal.

- **Publisher confirms** (`Confirm.Select` on the channel, then broker-to-client `Basic.Ack`/`Basic.Nack` keyed by a per-channel publish sequence number, with `multiple` batching) are the fix. A confirm means the broker has taken responsibility: written to disk for persistent messages on durable queues, replicated to a quorum for quorum queues, or merely enqueued for transient ones. **Unroutable messages are confirmed as acked** unless published `mandatory`, in which case `Basic.Return` arrives before the ack; a confirmed message is not the same as a delivered one.
- **`Tx.Select` / `Tx.Commit`** make the channel transactional: publishes and acks are batched until commit. Each commit is a synchronous round trip plus an fsync, which is why throughput drops by orders of magnitude and why the RabbitMQ docs steer everyone to confirms. Transactions are per channel and cover publishes and acks on that channel only; they are not distributed and not XA.

## Heartbeats and connection lifetime

The `heartbeat` value from `Connection.Tune` is a timeout in **seconds**; RabbitMQ proposes 60 (older releases proposed 580). The rules: each side sends a HEARTBEAT frame (type 8, channel 0, empty payload, 8 bytes on the wire) when it has sent nothing for timeout/2; a peer that sees **no frames of any kind** for two timeouts treats the connection as dead. Any frame resets the receiver's clock, so a busy connection never carries a heartbeat frame. Heartbeats are sent at the protocol layer by the client library's I/O thread, which means:

- A client whose heartbeat thread is blocked (a `synchronized` consumer callback that also publishes on the same channel, a GC pause longer than 2 x timeout) is disconnected by the broker with `Connection.Close 320 CONNECTION_FORCED` or a bare TCP reset, and its unacked messages are redelivered elsewhere while it is still processing them. Duplicate side effects follow.
- A middlebox idle timeout (NAT, load balancer, cloud firewall: typically 350 s to 15 min) shorter than 2 x heartbeat silently drops the connection; the client finds out on the next publish, the consumer side may never find out. Pick a heartbeat well under half the smallest idle timeout on the path.
- `heartbeat=0` from either side disables them. Do not; it also disables the one mechanism that detects a half-open TCP connection.

A connection closes cleanly with `Connection.Close` (reply-code, reply-text, failing class/method ids) answered by `Close-Ok`; the codes are the HTTP-like ones already seen (`320 CONNECTION_FORCED` admin or heartbeat kill, `501 FRAME_ERROR`, `503 COMMAND_INVALID`, `504 CHANNEL_ERROR`, `505 UNEXPECTED_FRAME`, `530 NOT_ALLOWED`, `540 NOT_IMPLEMENTED`, `541 INTERNAL_ERROR`) versus the channel-scoped ones (`403 ACCESS_REFUSED`, `404 NOT_FOUND`, `405 RESOURCE_LOCKED` for an exclusive queue, `406 PRECONDITION_FAILED`, `311 CONTENT_TOO_LARGE`, `312 NO_ROUTE` on a `Basic.Return`, `313 NO_CONSUMERS`). Automatic reconnection is a library feature, not a protocol one: the Java client re-opens the connection, channels and consumers but **does not replay unconfirmed publishes**.

## AMQP 1.0 in one page

Everything above is 0-9-1. AMQP 1.0 is a different design that kept the name:

- **Protocol header** `AMQP` + protocol-id (`0` raw, `2` TLS, `3` SASL) + `1 0 0`. TLS and SASL are layered as separate headers exchanged in sequence, not negotiated inside a method.
- **Frame**: 8-byte header `size (4, includes the header) | DOFF (1, header length in 4-byte words) | type (1: 0 AMQP, 1 SASL) | channel (2)`, then an extended header and a **performative** encoded as a described type. Frames are typed by the performative, not a frame-type byte.
- **Performatives**: `open` (container-level: max-frame-size, channel-max, idle-time-out, offered/desired capabilities), `begin` (session), `attach` (a unidirectional **link** from a source to a target "node", with settle modes), `flow` (credit), `transfer` (a message or a fragment of one), `disposition` (settle deliveries: `accepted`, `rejected`, `released`, `modified`), `detach`, `end`, `close`. Containers hold sessions, sessions hold links, links carry transfers.
- **Flow control is credit-based and receiver-driven.** A receiver grants N link credits with `flow`; a sender may send at most N transfers until more arrive. This replaces 0-9-1's prefetch, is also what Azure Service Bus and Event Hubs clients expose as "prefetch count", and stalls silently when the receiver forgets to top up.
- **Message format**: a sequence of sections, each an optional described type: `header` (durable, priority, ttl, delivery-count), `delivery-annotations`, `message-annotations` (broker-specific, where Azure puts enqueued time and sequence number and ActiveMQ puts its own ids), `properties` (message-id, to, subject, reply-to, correlation-id, content-type, absolute-expiry-time, group-id ...), `application-properties` (your headers), **body** as `data` (binary), `amqp-sequence` or `amqp-value`, then `footer`. The **bare message** (properties, application-properties, body) is what end-to-end signatures cover; annotations are mutable in transit.
- **Self-describing types.** Every value carries a type code (`0x40` null, `0x41` true, `0xa1` short UTF-8 string, `0xb1` long string, `0x70` uint, `0x83` timestamp ms, `0x98` UUID, `0xc0` list, `0xd1` map ...), with described types (`0x00` + descriptor + value) for performatives and sections. Wireshark decodes it; reading it by eye is not realistic.
- **No routing model.** Exchanges and queues are RabbitMQ/0-9-1 concepts; in 1.0 you attach to an address string and the broker decides what it means (`/queues/x` and `/exchanges/x/key` on RabbitMQ 4; queue or topic names on Artemis; entity paths on Azure). Consequently a 1.0 client library cannot declare a queue in a portable way; provisioning is a broker API or a management UI.
- **Settlement replaces ack**: a transfer is settled by the sender (at-most-once, `settled=true`), by the receiver (at-least-once, receiver sends `disposition accepted` and the sender settles), or both (exactly-once in spec terms, rarely implemented).

Interop summary: **AMQP 0-9-1 and 1.0 are different protocols**; the Azure "AMQP" SDKs will not connect to RabbitMQ 3.x without the 1.0 plugin, and Pika/amqplib will not connect to Artemis or Service Bus at all. RabbitMQ 4.x speaks both natively on the same port and converts between them, so a 0-9-1 publisher can feed a 1.0 consumer, with properties mapped as documented in the RabbitMQ release notes.

## Neighbours it is confused with

- **AMQP 0-10**: a third, incompatible dialect used by the old Qpid C++ broker. Effectively dead.
- **JMS / Jakarta Messaging** is a Java **API**, not a wire protocol. A JMS client talks to its broker in whatever the broker speaks (OpenWire for ActiveMQ Classic, Artemis Core, AMQP 1.0 via Qpid JMS). "JMS over AMQP" means Qpid JMS.
- **MQTT** is a separate pub/sub protocol (TCP 1883/8883, 2-byte fixed header, topics with `+`/`#`, QoS 0/1/2, retained messages, no queues). RabbitMQ and Artemis speak it too, which does not make it AMQP.
- **STOMP** is a text protocol (TCP 61613) with HTTP-shaped frames; the lowest-common-denominator way for a shell script or a WebSocket to reach a broker.
- **Kafka** has its own binary protocol on 9092 and a log model (partitions, offsets, retention) rather than queues. The difference that matters is in `async-systems`.
- **AMQP over WebSocket**: AMQP 1.0 frames in binary WebSocket messages, subprotocol `amqp`, used by Azure on port 443 to escape corporate egress rules (`AmqpWebSockets` transport). RabbitMQ offers Web-STOMP and Web-MQTT but not Web-AMQP.

## Observing it

- `tcpdump -nn port 5672 -w amqp.pcap` and open in Wireshark: the `amqp` dissector decodes 0-9-1 and 1.0, and `amqp.method.class == 60 && amqp.method.method == 40` isolates publishes. On 5671 you see TLS records only; capture with the client's `SSLKEYLOGFILE` support or on the broker side after termination.
- The `netcat` header probe above proves the port, the version and the product in one line, before any credentials.
- `rabbitmqctl list_connections name client_properties recv_oct send_oct state` and `list_channels connection prefetch_count messages_unacknowledged` show what each client negotiated and whether prefetch is actually set; the management UI (TCP 15672) shows the same with the client library name and version from `Connection.Start-Ok`.
- `rabbitmq-diagnostics check_port_connectivity` and the `/api/aliveness-test/<vhost>` endpoint do an end-to-end declare/publish/consume.

## What this means for AEM

- **AEM has no AMQP in it.** Replication, Sling jobs, Sling eventing and (on AEM as a Cloud Service) Sling Content Distribution all run on HTTP and the repository; Adobe I/O Events delivers by webhook or journaling over HTTPS. Any AMQP in an AEM solution is custom OSGi code with a client library (`com.rabbitmq:amqp-client` ships as an OSGi bundle; Qpid JMS or the Azure SDKs for 1.0) and should be designed as one long-lived connection per instance with a channel per thread, not a connection per request. On Cloud Service, remember that every publish pod is a separate client, so a consumer inside AEM becomes N competing consumers and inbound ports do not exist at all: AEM can only ever be the client.
- **Egress on AEM as a Cloud Service is HTTP-shaped by default.** Outbound traffic is limited to ports 80 and 443 unless the program has advanced networking; reaching 5671/5672 needs flexible port egress or VPN, and with flexible port egress non-HTTP traffic must be routed through the Adobe-provided proxy (the `AEM_PROXY_HOST` / `AEM_PROXY_PORT` variables, SOCKS for non-HTTP), which the AMQP client must be configured to use. The practical workaround is a broker that offers AMQP over WebSocket on 443 or an HTTP ingestion front, which is also what makes the heartbeat and idle-timeout discussion above matter: the Adobe egress path has its own idle limits.
- **Treat the broker like any other outbound dependency**: connect and publish on a bounded timeout, never on the request thread, and never with the default unlimited prefetch on a consumer that writes to the JCR, or a burst of messages becomes a burst of concurrent sessions and `OakMerge` conflicts. The patterns are in `reliability-patterns` and `async-systems`.

## Cheat sheet

| | AMQP 0-9-1 | AMQP 1.0 |
|---|---|---|
| Header | `AMQP 0 0 9 1` | `AMQP 0 1 0 0` (`2` TLS, `3` SASL) |
| Ports | 5672 / 5671 TLS | same |
| Frame | `type(1) channel(2) size(4) payload 0xCE` | `size(4) doff(1) type(1) channel(2) performative` |
| Multiplexing | channels on a connection | sessions and links in a container |
| Routing | exchange + binding + routing key in the protocol | broker-defined address, nothing in the spec |
| Message | METHOD + HEADER (14 properties) + BODY frames | sections: header, annotations, properties, app-properties, body, footer |
| Throttle | `Basic.Qos` prefetch (0 = unlimited, `global` scope differs on RabbitMQ) | receiver-granted link credit |
| Delivery guarantee | ack/nack/reject by channel-scoped delivery-tag; publisher confirms | settlement + disposition (accepted/rejected/released/modified) |
| Liveness | heartbeat seconds from `Tune`, dead after 2 missed | `idle-time-out` ms in `open`, empty frames |
| Speakers | RabbitMQ and its clients | Azure Service Bus / Event Hubs, ActiveMQ, Artemis, Qpid, Solace, IBM MQ, RabbitMQ 4 |
