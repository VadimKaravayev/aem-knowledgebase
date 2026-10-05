# Caching — CDN, Redis-style caches, invalidation, cache-aside

General system-design reference (not AEM-specific), with pointers to the AEM instances of each pattern.

## Where caches live (the layers)

Client/browser → **CDN edge** → reverse proxy / app-level cache → **distributed cache (Redis/Memcached)** → DB internal caches. Each layer has different invalidation reach: you control the CDN and Redis directly; browser caches you only control *in advance* via headers (once a long `max-age` ships, you can't take it back — hence cache-busting file names for static assets: content hash in the filename, `immutable` + 1-year TTL).

## HTTP/CDN caching

- `Cache-Control` decides policy: `max-age` (freshness), `s-maxage` (shared caches only), `public/private`, `no-cache` (= **revalidate**, not "don't store" — that's `no-store`), `stale-while-revalidate` (serve stale, refresh in background — cheap latency win), `stale-if-error` (resilience: serve stale when origin is down).
- Revalidation: `ETag`/`If-None-Match` → `304` saves the body, not the round trip. Weak vs strong ETags matter for range requests.
- `Vary` is the footgun: `Vary: Accept-Encoding` is fine; `Vary: Cookie` or `Vary: User-Agent` shatters the cache into near-useless fragments (hit rate → 0 while dashboards say "cache enabled").
- CDN invalidation is either **purge by URL** (exact, slow to enumerate), **purge by surrogate key/tag** (tag responses with content IDs at origin, purge the tag — the scalable answer), or **purge all** (the 3am answer).
- AEM instance: the Dispatcher's `.stat`-file invalidation and the "Invalidate vs Flush" distinction (≤ 100 paths, flush for HTML clientlibs) — see [cloud-manager-ams-production-pipelines](../cloud-service/cloud-manager-ams-production-pipelines.md); AEMaaCS layers Adobe's Fastly CDN *on top of* Dispatcher, so there are **two caches to purge** — same trap as Edge Functions' fetch cache in [edge-functions](../cloud-service/edge-functions.md).

## Cache-aside and its siblings

- **Cache-aside (lazy)**: app reads cache → miss → reads DB → writes cache with TTL. Default pattern; cache failure degrades to DB load, not errors. Weaknesses: first-request latency, stampede on hot-key expiry, and a **stale-write race**: reader A misses, reads DB (old value); writer B updates DB and invalidates; A then writes its stale value into the cache, where it lives until TTL. Mitigations: short TTLs as backstop, compare-and-set/versioned values, or delete-then-short-delay-delete-again.
- **Read-through / write-through**: the cache itself loads/stores (cache library or sidecar owns the DB call). Write-through keeps cache fresh but adds write latency.
- **Write-behind**: ack on cache write, flush to DB async — fast, but now the cache is the system of record until flush; a crash loses writes. Rarely worth it outside counters.
- Rule of thumb: **invalidate (delete), don't update** the cache on writes — updates reintroduce ordering races; a delete forces the next reader to load the truth.

## Stampedes, hot keys, and other failure modes

- **Thundering herd / dogpile**: popular key expires, 10k concurrent requests all miss and hit the DB. Fixes: per-key mutex/single-flight (one loader, others wait or serve stale), **probabilistic early refresh** (refresh before expiry with probability rising as TTL nears), jittered TTLs so keys don't expire in sync.
- **Cache penetration**: requests for keys that don't exist bypass the cache every time (nothing to cache). Fix: cache negative results with a short TTL, or a Bloom filter in front.
- **Cache avalanche**: cache node restarts cold → all traffic lands on the DB at once. Fix: warm-up, request coalescing, and sizing the DB to survive ~0% hit rate briefly — or admit the cache is load-bearing and treat it as tier-1 infrastructure.
- **Hot key**: one key exceeds a single cache node's capacity. Fix: local/in-process L1 cache in front of Redis, or key replication (`key#1..N`, random read).

## Redis-style distributed caches

- Memcached = pure cache (multi-threaded, LRU, no persistence). Redis = data structures (hashes, sorted sets, streams), optional persistence/replication, Lua for atomic multi-step ops, pub/sub — which is why "cache" Redis instances accrete into databases; decide upfront whether loss of the instance is an incident or a non-event, and keep the two uses on separate instances.
- Eviction policy matters: `allkeys-lru` for a pure cache; `noeviction` (default!) makes a full cache **reject writes** instead of evicting — a classic production surprise.
- TTL everything. A cache where only *some* keys expire becomes a leak with a hit rate.
- Serialization cost is part of the latency budget; caching a 2 MB JSON blob per request can be slower than the query it replaced.

## What to cache (and not)

Good: read-heavy + tolerant of bounded staleness (rendered fragments, config, session-adjacent lookups, expensive aggregations). Bad: anything where staleness = correctness bug (authorization decisions, balances, inventory at checkout) unless paired with revalidation at the point of commitment. State the allowed staleness per data class **in writing** — "cache it" without a staleness budget is where incidents come from.

## Design-review checklist

TTL + jitter on every key; stampede plan for hot keys; invalidation path tested (not just expiry); negative caching; metrics for hit rate *and* origin load (see [[observability]]); behavior verified with the cache **empty and down** (the cache is an optimization — if the system can't survive without it, it's a dependency and must be engineered as one, see [[reliability-patterns]]).
