# Sharding — splitting data across nodes

How a dataset too big or too write-heavy for one machine is divided, how keys are mapped to shards, and how four well-known systems do it. Docs-based; the Java samples were compiled and run on Java 23. Related: [[scalability]] (stateless tiers and load balancers), [[data-storage]] (replication topologies), [[consistency-models]] (what readers see), [[autoscaling]] (adding compute automatically).

## Scaling up, scaling out

- **Scaling up (vertical)**: a bigger machine. No architecture change, no new failure modes. Limits: a hard ceiling, price that grows faster than capacity near the top, and still one machine to lose.
- **Scaling out (horizontal)**: more machines. Trivial for stateless services (any node serves any request); hard for data, because the data has to be *placed* — either copied to every node (replication) or divided between them (sharding).
- The usual order for a database: index and cache → scale up → add read replicas → partition tables inside one instance → shard. Each step is cheaper and more reversible than the next. Sharding is last because it is the only one that changes what queries are possible.

## Replication vs sharding

| | Replication | Sharding |
|---|---|---|
| What each node holds | The **same** data | **Different** data |
| Scales | Reads | Writes and total data size |
| Availability | Improves (a copy survives) | Worsens by itself (more nodes, each a partial outage) |
| Main cost | Replication lag, failover | Cross-shard queries and transactions, rebalancing |

They solve different problems and are combined in practice: the dataset is cut into shards, and **each shard is a replica set**. A read replica never helps a write bottleneck — every replica must still apply every write.

## Choosing where a key goes

### Range-based

Each shard owns a contiguous range of the key (`a–g`, `h–p`, `q–z`), kept in a sorted map of range boundaries.

- Good: **range scans stay on one or a few shards** (`WHERE created BETWEEN …`, prefix scans). Ranges can be split and merged automatically as they grow (HBase, Bigtable, CockroachDB, MongoDB ranged shard keys).
- Bad: **hot spots**. A monotonically increasing key (timestamp, auto-increment ID) sends every new write to the last shard. Uneven key popularity gives uneven shards.
- Fix for monotonic keys: prefix with something well spread (tenant, hash bucket) so the range is `(bucket, timestamp)`.

### Hash-based

`shard = hash(key) mod N`.

- Good: uniform spread regardless of key shape; no hot last shard.
- Bad: **range scans become scatter-gather** over all shards, and **changing N remaps almost every key** — going from 4 to 5 shards moves about 80% of the data (measured below). Resizing is a full migration.
- Standard fix for resizing: hash into a **fixed number of logical partitions** much larger than the node count (1,024, 16,384), and keep a small partition → node table. Growing the cluster moves whole partitions; no key ever changes partition. Redis Cluster's slots and Kafka's partitions are this design.

### Consistent hashing

Nodes and keys are hashed onto the same circular space (a ring). A key belongs to the **first node clockwise** from its position.

- Adding or removing a node moves only the keys between it and its neighbour — about `1/N` of the data instead of nearly all of it.
- **Virtual nodes** are not optional: with one position per node the arcs are wildly uneven (measured below: one node with 3% of keys, another with 41%), and a dead node's whole load lands on a single neighbour. Giving each node 100–200 positions evens the load and spreads a failure across all survivors. More positions for a bigger machine gives weighted distribution.
- Replication falls out naturally: store each key on the next R *distinct physical* nodes clockwise.
- Variants worth knowing by name: **rendezvous (highest-random-weight) hashing** — no ring, score every node per key and take the top; **jump consistent hash** — no memory at all, but buckets can only be added or removed at the end, so it suits numbered shards rather than arbitrary node failures; **bounded-load** consistent hashing — caps any node at a factor above average, for load balancers.

### Directory-based

A lookup table maps a key (often a tenant) to its shard. Most flexible — a single large tenant can be moved alone — at the cost of an extra lookup and a component that must itself be highly available.

### Summary

| Strategy | Range queries | Even load | Adding a node |
|---|---|---|---|
| Range | Cheap | Only with a well-spread key | Split a range |
| Hash mod N | Scatter-gather | Yes | Remaps nearly everything |
| Fixed partitions | Scatter-gather | Yes | Move whole partitions |
| Consistent hashing | Scatter-gather | Yes, with virtual nodes | Moves about `1/N` |
| Directory | Depends on layout | Manual | Edit the table, move data |

Many stores combine two: a **hashed partition key** picks the shard and a **sorted key within the partition** gives range scans inside it (DynamoDB sort key, Cassandra clustering columns).

## How real systems do it

### Amazon DynamoDB

- The **partition key is hashed** to choose a partition; an optional **sort key** orders items inside it. Queries need the partition key; anything else is a full-table scan or a secondary index.
- Partitions are split automatically by size and throughput, and each is replicated across three availability zones. The application never sees shards.
- The design trap is the **hot partition**: throughput is divided among partitions, so a low-cardinality or skewed partition key (status, date, one huge tenant) throttles while the table as a whole is underused. Adaptive capacity softens it; a better key fixes it.
- Naming note: the Dynamo *paper* described a consistent-hashing ring with virtual nodes and leaderless quorums. DynamoDB the service borrows the name and the partitioning idea, not that architecture.

### Apache Cassandra

- The textbook consistent-hashing ring: the partition key is hashed (Murmur3) to a 64-bit **token**; each node owns token ranges; **vnodes** give every node many small ranges (`num_tokens`, 16 by default since 4.0).
- Replication factor R places each partition on R nodes walking the ring, rack- and datacenter-aware. Consistency is chosen per query (`ONE`, `QUORUM`, `ALL`).
- Leaderless: any node accepts any request and forwards it. Drivers are token-aware and send the request straight to a replica.
- Trap: the order-preserving partitioner makes range scans over keys possible and recreates every range-sharding hot spot; it is discouraged. Model tables per query instead.

### Akamai CDN

- Consistent hashing was invented for this problem: assigning web objects to cache servers so that a server joining or leaving does not invalidate every cache in the cluster. The 1997 paper's authors went on to found Akamai.
- Two levels: a global step (DNS-based) picks a cluster near the user; within the cluster, consistent hashing picks the server for a given piece of content, so the same object keeps hitting the same warm cache.
- The lesson generalises to any cache tier: with `hash mod N`, losing one node turns the whole fleet cold at once.

### Redis Cluster

- **Not consistent hashing**: a fixed **16,384 hash slots**, `slot = CRC16(key) mod 16384`, each slot assigned to one primary. Resharding moves slots between nodes while the cluster serves traffic.
- Clients cache the slot map; a node that does not own a key answers `MOVED` (map changed, update it) or `ASK` (slot is mid-migration, ask there once).
- **Multi-key commands and transactions only work inside one slot**, otherwise `CROSSSLOT`. A **hash tag** forces keys together: only the part in braces is hashed, so `{user:42}:cart` and `{user:42}:profile` share a slot. Overusing one tag rebuilds a hot shard by hand.
- Each primary has replicas with asynchronous replication, so a failover can lose acknowledged writes. More in [[redis]].

## Java samples

No sharding support exists in the JDK; applications either route themselves or use a client that does. The first three samples are the routing logic in plain Java.

### Hash-mod routing, and the negative-hash trap

```java
static int shardFor(String key, int shards) {
    return Math.floorMod(hash(key), shards);   // never negative
}
```

- `key.hashCode() % n` is negative for about half of all keys. `Math.abs(hashCode()) % n` is the usual fix and is still wrong: `Math.abs(Integer.MIN_VALUE)` is negative, and real strings hash to it (`"polygenelubricants".hashCode()` does). Use `Math.floorMod`.
- Do not use `Object.hashCode()` for routing at all: it is not stable across JVM runs for most classes, and `String.hashCode()` spreads poorly. Use a real hash function (the `hash` below, or Murmur3 from a library) so every service instance, in every language, agrees.

### Consistent-hash ring with virtual nodes

```java
import java.nio.ByteBuffer;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.Map;
import java.util.NavigableMap;
import java.util.TreeMap;

public final class HashRing<N> {
    private final NavigableMap<Long, N> ring = new TreeMap<>();
    private final int vnodes;

    public HashRing(int vnodes) { this.vnodes = vnodes; }

    public void add(N node)    { for (int i = 0; i < vnodes; i++) ring.put(hash(node + "#" + i), node); }
    public void remove(N node) { for (int i = 0; i < vnodes; i++) ring.remove(hash(node + "#" + i)); }

    public N nodeFor(String key) {
        if (ring.isEmpty()) throw new IllegalStateException("no nodes");
        Map.Entry<Long, N> e = ring.ceilingEntry(hash(key));     // first node clockwise
        return (e != null ? e : ring.firstEntry()).getValue();   // wrap around the ring
    }

    static long hash(String s) {
        try {
            byte[] d = MessageDigest.getInstance("MD5").digest(s.getBytes(StandardCharsets.UTF_8));
            return ByteBuffer.wrap(d).getLong();                 // first 8 bytes
        } catch (NoSuchAlgorithmException e) { throw new IllegalStateException(e); }
    }
}
```

Measured with 100,000 keys, growing from 4 to 5 nodes:

| Routing | Keys that changed node | Load per node before the change |
|---|---|---|
| `hash mod N` | 79.9% | even |
| Ring, 1 position per node | 20.6% | 3% – 41% |
| Ring, 150 positions per node | 18.2% | 22% – 31% |

The ring is not thread-safe as written: guard it with a lock, or build a new immutable ring on each membership change and swap the reference. MD5 is used here only as a well-spread hash that ships with the JDK, not for security.

### Range routing

```java
NavigableMap<String, String> ranges = new TreeMap<>(Map.of(
        "",  "shard-a",     // "" .. "h"
        "h", "shard-b",     // "h" .. "q"
        "q", "shard-c"));   // "q" .. end

String shard = ranges.floorEntry("miller").getValue();   // shard-b
```

The map holds each range's lower bound; `floorEntry` finds the range a key falls in. Splitting a range is one `put`.

### Libraries that do the routing

- **Guava** — `Hashing.consistentHash(hashCode, buckets)` is jump consistent hash: returns a bucket index, moves about `1/N` of keys when `buckets` grows by one, and only supports adding or removing the last bucket.
- **Spring** — `AbstractRoutingDataSource`: override `determineCurrentLookupKey()` to return the shard for the current request (typically read from a thread-bound context set by a filter), and it hands out a connection from the matching `DataSource`. It routes; it does not split queries or join across shards.
- **Apache ShardingSphere-JDBC** — a JDBC driver that parses SQL, routes by configured sharding rules, and merges results from several shards. The option when the application should keep writing ordinary SQL.
- **Redis** — Jedis `JedisCluster` and Lettuce's cluster client compute the slot and follow `MOVED`/`ASK` themselves.
- **Cassandra** — the Java driver's default load-balancing policy is token-aware.
- **Kafka** — the producer's default partitioner is `murmur2(key) mod partitions`: plain hash-mod, which is why **adding partitions to a topic breaks per-key ordering** for existing keys.

## What sharding costs

- **The shard key is nearly permanent.** It decides which queries are single-shard; changing it is a full re-migration. Pick it from the dominant access pattern, with high cardinality and even traffic.
- **Cross-shard queries** fan out to every shard and are as slow as the slowest one; sorting, paging and aggregating across shards is application work.
- **Cross-shard transactions** need two-phase commit or sagas. Design so that one business invariant lives inside one shard.
- **Global uniqueness and IDs**: an auto-increment column is per shard. Use IDs that are unique without coordination (UUIDv7, Snowflake-style time + node + sequence).
- **Hot keys** survive any hashing scheme — one celebrity account is one key on one shard. Handle them specifically (dedicated shard, key splitting, caching).
- **Rebalancing** moves data under live traffic and competes with it for I/O. Throttle it and rehearse it.
- **Operations multiply**: backups, schema changes and upgrades run per shard, and a restore to a consistent point across shards is its own problem.

## What this means for AEM

- **AEM replicates; it does not shard.** A publish farm is N full copies of the content, which scales reads and nothing else. There is no supported way to split one repository's content tree across nodes.
- **AEM 6.5 on MongoMK** uses MongoDB replica sets; MongoDB sharding is not a supported configuration for the AEM repository. Irrelevant on AEMaaCS, where the storage layer is Adobe-managed.
- When a content repository outgrows one author, the answer is partitioning by site or business unit into separate author instances — directory-style sharding done by hand ([multi-author-scaling](../infrastructure-ops/multi-author-scaling.md)).
