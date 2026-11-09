# Redis: how it works, when to choose it, alternatives

Redis is an **in-memory data structure server**: a key-value store where values are not just strings but lists, hashes, sets, sorted sets, streams and more, each with its own atomic commands. Because everything lives in RAM, most operations take well under a millisecond. It's most often used as a cache, a session store, a rate limiter, a leaderboard or a lightweight queue next to a main database, rarely as the main database itself.

## How it works

### Keys, values and data types

Every piece of data is a **key** (a string, conventionally namespaced with colons, like `user:42:profile`) pointing to a value of one type:

| Type | What it is | Typical use | Example |
|---|---|---|---|
| **String** | text, number or bytes, up to 512 MB | cache entries, counters, flags | `SET page:home "<html>…" EX 60` · `INCR views:post:7` |
| **Hash** | field → value map inside one key | objects, sessions | `HSET session:abc user_id 42 role admin` |
| **List** | ordered list, push/pop at both ends | simple queues, recent items | `LPUSH jobs "job-1"` · `BRPOP jobs 0` |
| **Set** | unordered unique values | tags, "who has seen this", dedupe | `SADD online:users 42` |
| **Sorted set** | unique values, each with a score, kept sorted | leaderboards, priority queues, time windows | `ZADD scores 1500 player:7` · `ZREVRANGE scores 0 9` |
| **Stream** | append-only log with IDs and consumer groups | durable-ish event queues | `XADD orders * id 1001` · `XREADGROUP …` |
| **Bitmap / HyperLogLog** | bits; approximate unique count in 12 KB | daily active users, feature flags | `PFADD visitors:today 42` · `PFCOUNT visitors:today` |
| **Geo** | coordinates in a sorted set | "nearest shops" | `GEOADD shops 0.1 0.2 shop:1` · `GEOSEARCH …` |

JSON documents, full-text and vector search, and time series come as modules (Redis Stack), included in the main distribution since Redis 8.

**Expiry (TTL):** any key can expire on its own: `SET otp:42 913442 EX 300` disappears after 5 minutes. This is what makes Redis a natural cache.

### One thread runs the commands

Redis executes commands **one at a time, on a single thread**, from an event loop. Network I/O can use extra threads (Redis 6+), but command execution stays sequential.

- Every single command is **atomic**: `INCR` from two clients never loses a count. No locks are needed.
- Speed comes from RAM plus no locking, not from parallelism. One instance handles on the order of 100,000+ simple operations per second.
- The cost: **one slow command blocks everyone.** `KEYS *` on millions of keys, `SMEMBERS` on a huge set, or deleting a giant key stalls every other client. Use `SCAN` instead of `KEYS`, `UNLINK` instead of `DEL` for big keys, and keep values small.

For several commands as one unit:

- **`MULTI` … `EXEC`** queues commands and runs them together with nothing in between, but has **no rollback**: if one command fails, the others still apply. `WATCH key` adds optimistic locking (the `EXEC` is cancelled if the key changed).
- **Lua scripts / functions** (`EVAL`, `FCALL`) run logic atomically on the server, for read-check-write operations.
- **Pipelining** sends many commands without waiting for each reply; it saves round trips but isn't atomic.

### Memory limit and eviction

All data must fit in RAM. `maxmemory` sets the limit, and `maxmemory-policy` decides what happens when it's reached:

| Policy | Behaviour | Use for |
|---|---|---|
| `noeviction` (default) | writes fail with an error | data you can't lose silently |
| `allkeys-lru` / `allkeys-lfu` | drop least-recently / least-frequently used keys | a pure cache |
| `volatile-lru` / `volatile-lfu` / `volatile-ttl` | drop only keys that have a TTL | mixed cache + data in one instance |

A cache instance left on `noeviction` starts rejecting writes when full; a data instance on `allkeys-lru` silently loses data. Set the policy for the job.

### Persistence: optional and partial

Redis is in-memory first; writing to disk is optional:

- **RDB snapshots**: a full copy saved every N minutes or N changes. Compact and fast to restart from, but you lose everything since the last snapshot. Saving forks the process; on a busy instance changed pages are copied, so memory use can briefly grow a lot.
- **AOF (append-only file)**: every write is logged. With the default `appendfsync everysec`, a crash loses up to about one second of writes; `always` loses nothing but is much slower.
- Both can be on; or neither, for a pure cache.

### Replication and scaling

- **Replicas** copy a primary **asynchronously**. If the primary fails right after acknowledging a write, that write can be lost on failover. `WAIT` reduces the window but doesn't make Redis strongly consistent.
- **Sentinel** watches primaries and promotes a replica on failure (high availability, no sharding).
- **Redis Cluster** shards keys over **16,384 hash slots** across several primaries. Commands touching several keys (`MGET`, `MULTI`, Lua) work only if all keys are in the same slot; force that with a hash tag: `{user:42}:cart` and `{user:42}:profile` land together.

## Common patterns

**Cache-aside** (the most common use):

```text
value = GET product:7
if value is missing:
    value = load from database
    SET product:7 value EX 300      # TTL bounds how stale it can get
return value
```

On update, write to the database, then `DEL product:7`. Add a little random jitter to TTLs so many keys don't expire at the same moment and stampede the database.

**Rate limiting** (fixed window, 100 requests per minute):

```text
count = INCR rate:user:42:<current-minute>
if count == 1: EXPIRE rate:user:42:<current-minute> 60
if count > 100: reject
```

**Simple lock** (one worker at a time):

```text
SET lock:report:daily <random-token> NX PX 30000    # only if absent, auto-release after 30 s
... do the work ...
release only if the value is still <random-token>   # Lua script, so you don't free someone else's lock
```

Fine for avoiding duplicate work. **Not safe for correctness** (money, exactly-once): a paused process can outlive its lock. Use a database row lock, or fencing tokens with a coordination service such as etcd or ZooKeeper.

**Leaderboard**: `ZINCRBY leaderboard 10 player:7`, then `ZREVRANGE leaderboard 0 9 WITHSCORES` for the top ten and `ZREVRANK leaderboard player:7` for one player's position, each fast even with millions of players.

**Queues**: lists (`LPUSH` + `BLMOVE` to a processing list) for simple jobs; **streams** with consumer groups when you need acknowledgements, retries and several consumers. **Pub/Sub** is fire-and-forget: subscribers that are offline miss the message entirely.

## Java examples

The two main Java clients are **Jedis** (simple, synchronous; used below, version 5.x) and **Lettuce** (async and reactive, thread-safe single connection; the default in Spring Data Redis). Redisson is a third option that adds ready-made locks, maps and queues on top of Redis.

```xml
<dependency>
  <groupId>redis.clients</groupId>
  <artifactId>jedis</artifactId>
  <version>5.1.0</version>
</dependency>
```

### Connecting

`JedisPooled` keeps a connection pool inside and is thread-safe: create **one per application** and share it, never one per request.

```java
import redis.clients.jedis.DefaultJedisClientConfig;
import redis.clients.jedis.HostAndPort;
import redis.clients.jedis.JedisPooled;

JedisPooled redis = new JedisPooled(
    new HostAndPort("redis.example.com", 6379),
    DefaultJedisClientConfig.builder()
        .password(System.getenv("REDIS_PASSWORD"))   // never hard-code
        .ssl(true)
        .timeoutMillis(2000)                         // fail fast instead of hanging threads
        .build());

// local development: new JedisPooled("localhost", 6379)
```

### Cache-aside with TTL and jitter

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import redis.clients.jedis.params.SetParams;
import java.util.concurrent.ThreadLocalRandom;

public class ProductCache {
    private static final int BASE_TTL_SECONDS = 300;
    private final JedisPooled redis;
    private final ProductRepository db;
    private final ObjectMapper json = new ObjectMapper();

    public ProductCache(JedisPooled redis, ProductRepository db) {
        this.redis = redis;
        this.db = db;
    }

    public Product get(long id) throws Exception {
        String key = "product:" + id;
        String cached = redis.get(key);
        if (cached != null) {
            return json.readValue(cached, Product.class);        // cache hit
        }
        Product product = db.findById(id);                        // cache miss: load from the database
        if (product != null) {
            int ttl = BASE_TTL_SECONDS + ThreadLocalRandom.current().nextInt(60);  // jitter
            redis.set(key, json.writeValueAsString(product), SetParams.setParams().ex(ttl));
        }
        return product;
    }

    public void update(Product product) {
        db.save(product);                     // 1. write the source of truth
        redis.del("product:" + product.id()); // 2. drop the stale copy; the next read refills it
    }
}
```

If Redis is down, `get` throws. For a cache, catch the exception and fall back to the database so a cache outage doesn't become a site outage.

### Rate limiting

```java
public boolean allowRequest(String userId) {
    long minute = System.currentTimeMillis() / 60_000;
    String key = "rate:" + userId + ":" + minute;
    long count = redis.incr(key);
    if (count == 1) {
        redis.expire(key, 60);       // first request in this window starts the timer
    }
    return count <= 100;             // 100 requests per minute
}
```

If the process dies between `incr` and `expire`, the key never expires. For a strict version, do both in one Lua script.

### Lock with safe release

```java
import java.util.List;
import java.util.UUID;

private static final String RELEASE_SCRIPT =
    "if redis.call('get', KEYS[1]) == ARGV[1] then " +
    "  return redis.call('del', KEYS[1]) " +
    "else return 0 end";

public void runDailyReportOnce() {
    String key = "lock:report:daily";
    String token = UUID.randomUUID().toString();

    String ok = redis.set(key, token, SetParams.setParams().nx().px(30_000));
    if (!"OK".equals(ok)) {
        return;                                   // another worker holds the lock
    }
    try {
        generateReport();
    } finally {
        // delete only if it's still our token, so we never release someone else's lock
        redis.eval(RELEASE_SCRIPT, List.of(key), List.of(token));
    }
}
```

Good for "don't run the job twice"; not for correctness-critical work (see the lock pattern above).

### Leaderboard with a sorted set

```java
import redis.clients.jedis.resps.Tuple;

redis.zincrby("leaderboard", 10, "player:7");                 // add 10 points

List<Tuple> top10 = redis.zrevrangeWithScores("leaderboard", 0, 9);
for (Tuple t : top10) {
    System.out.println(t.getElement() + " " + (long) t.getScore());
}

Long rank = redis.zrevrank("leaderboard", "player:7");        // 0 = first place, null = not ranked
```

### Session or object as a hash

```java
import java.util.Map;

redis.hset("session:abc", Map.of("userId", "42", "role", "admin"));
redis.expire("session:abc", 1800);                            // 30-minute session

Map<String, String> session = redis.hgetAll("session:abc");   // empty map if expired
String role = redis.hget("session:abc", "role");
```

### Queue with a stream and a consumer group

```java
import redis.clients.jedis.StreamEntryID;
import redis.clients.jedis.exceptions.JedisDataException;
import redis.clients.jedis.params.XAddParams;
import redis.clients.jedis.params.XReadGroupParams;
import redis.clients.jedis.resps.StreamEntry;

// producer
redis.xadd("orders", XAddParams.xAddParams(), Map.of("orderId", "1001", "total", "12.50"));

// consumer: create the group once (ignore "already exists")
try {
    redis.xgroupCreate("orders", "billing", new StreamEntryID(0, 0), true);
} catch (JedisDataException e) {
    if (!e.getMessage().contains("BUSYGROUP")) throw e;
}

while (running) {
    List<Map.Entry<String, List<StreamEntry>>> batches = redis.xreadGroup(
        "billing", "worker-1",
        XReadGroupParams.xReadGroupParams().count(10).block(5000),  // wait up to 5 s
        Map.of("orders", StreamEntryID.UNRECEIVED_ENTRY));          // only new messages
    if (batches == null) continue;

    for (StreamEntry entry : batches.get(0).getValue()) {
        processOrder(entry.getFields());
        redis.xack("orders", "billing", entry.getID());             // done; won't be redelivered
    }
}
```

A message that is read but never acknowledged (the worker crashed) stays pending; another worker can take it over with `XAUTOCLAIM`.

### Spring Boot: caching with annotations

With `spring-boot-starter-data-redis` and `spring-boot-starter-cache`, Spring uses Redis behind `@Cacheable` with no Redis code in the service:

```properties
spring.data.redis.host=redis.example.com
spring.data.redis.port=6379
spring.data.redis.password=${REDIS_PASSWORD}
spring.data.redis.ssl.enabled=true
```

```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public RedisCacheConfiguration cacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(5))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));   // JSON, not Java serialization
    }
}

@Service
public class ProductService {
    @Cacheable(value = "products", key = "#id")     // reads Redis first, calls the method on a miss
    public Product find(long id) { return repository.findById(id).orElse(null); }

    @CacheEvict(value = "products", key = "#product.id")
    public Product save(Product product) { return repository.save(product); }
}
```

For direct access, inject `StringRedisTemplate`: `template.opsForValue().increment("views:post:7")`, `template.opsForZSet().incrementScore("leaderboard", "player:7", 10)`.

## Pros and cons

**Pros**

- Very fast: sub-millisecond reads and writes, high throughput from one small instance.
- Rich data structures with atomic operations: counters, sets, rankings and queues without writing concurrency code.
- Built-in expiry and eviction make caching trivial.
- Simple to run and to learn; client libraries in every language; a protocol (RESP) many other products copy.
- Flexible durability: from pure cache to AOF-on-every-write.

**Cons**

- **RAM-bound**: the whole dataset must fit in memory, which costs far more per GB than disk. Plan for 1.5–2× the data size to cover overhead and fork-time copying.
- **Weak durability and consistency**: snapshots lose recent writes, AOF `everysec` up to a second, and async replication can drop acknowledged writes on failover.
- **No query language**: you can only get data by key. "All users in country X" means you must have built a set for it yourself.
- **Single-threaded execution**: one slow command or one huge key stalls the instance; uses one CPU core per instance for commands.
- **Cluster limits**: multi-key operations only within one hash slot; resharding and hot keys need care.
- `MULTI` has no rollback; transactions are not relational transactions ([[database-transactions-isolation]]).
- **Licence change**: from Redis 7.4 the source licence is no longer BSD (RSALv2/SSPLv1, with AGPLv3 added as an option in Redis 8). That prompted the Valkey fork and changed what cloud providers offer.

## Deciding whether to use it

Redis is a good choice when **most** of these are true:

- The data is **hot**: read or changed very often, and latency matters (milliseconds or less).
- It **fits in RAM** comfortably, now and in a year.
- Losing it is **acceptable or recoverable**: it can be rebuilt from the main database (cache, derived data) or a short loss doesn't matter (rate-limit counters, presence, sessions that just log users out).
- Access is **by key**, or fits one of the data structures (counter, set, ranking, queue).
- You need something shared by **several app instances**. One instance alone may be better served by an in-process cache.

Think twice when:

- It would be the **only copy of important data** (orders, payments, user accounts). Use a real database; if you must, look at durable Redis-compatible services such as Amazon MemoryDB.
- You need **queries by anything other than the key**, reporting or joins.
- Data is **large and mostly cold**: paying for RAM to hold rarely-read data.
- You need **guaranteed delivery** of messages: use a message broker, or at least streams instead of Pub/Sub.
- You need a **correctness-critical lock**.

A useful test: *"If this Redis instance were wiped right now, what breaks and how do we recover?"* If the answer is "it refills from the database" or "users log in again", Redis fits. If the answer is "we lose orders", it doesn't.

Before adding it, check whether the existing database already covers the need. PostgreSQL handles modest job queues (`SELECT … FOR UPDATE SKIP LOCKED`), advisory locks, and caching via materialized views or unlogged tables. Redis earns its place when the load or latency outgrows that; every extra system is one more thing to run and keep in sync ([[database-data-models]]).

## Alternatives

| Alternative | What it is | Choose it when |
|---|---|---|
| **Valkey** | Linux Foundation fork of Redis 7.2, BSD licence, drop-in compatible | you want Redis with an open-source licence; now the default on several clouds |
| **Memcached** | pure in-memory cache, strings only, multithreaded, no persistence | you need only a simple, very large cache and nothing else |
| **Dragonfly** | Redis/Memcached-compatible, multithreaded, more memory-efficient (BSL licence) | one big instance must use many cores instead of a cluster |
| **KeyDB** | multithreaded Redis fork | similar goal to Dragonfly; check how active it is before adopting |
| **Garnet** | Microsoft research cache server speaking the Redis protocol, .NET | experimenting with high throughput in .NET-centric stacks |
| **Managed services** | AWS ElastiCache (Valkey/Redis OSS), Azure Managed Redis, Google Memorystore | you want Redis without running it yourself |
| **Amazon MemoryDB** | Redis/Valkey-compatible with a durable multi-zone transaction log | you need Redis's API *and* real durability |
| **Hazelcast / Apache Ignite** | in-memory data grids, compute next to data, Java-first | JVM systems needing distributed maps with richer querying |
| **etcd / ZooKeeper** | strongly consistent small key-value stores | locks, leader election and config that must be correct |
| **RabbitMQ / Kafka** | message brokers | queues and events that must not be lost ([[amqp]]) |
| **DynamoDB** | managed durable key-value/document database | key-based access at scale where data must persist |
| **In-process cache** (Caffeine, Guava, an LRU map) | cache inside the application's memory | a single instance, or per-instance caching is acceptable; no network hop at all |
| **PostgreSQL** | the database you may already have | modest queues, locks and caching before adding a new system |
