# URL shortener — the classic design problem, and where it really breaks

General system-design reference (not AEM-specific), with pointers to the AEM instances of each pattern. It is the standard interview warm-up because it touches every other doc in this folder. The hard parts are not the hashing. They are redirect semantics, cache behavior, ID allocation under concurrency, and abuse.

## Requirements and the shape of the load

- Functional: `POST /links {url, alias?, expiresAt?}` → short code; `GET /{code}` → redirect; optional custom aliases, expiry, per-link analytics, deletion/disable.
- Non-functional: redirect is the hot path. It must be **fast (p99 tens of ms), highly available, and read-dominated**, with reads to writes at about 100:1 or more. Creation can be slower and can be less available than redirects. A shortener that can't redirect is down. One that can't create links is degraded.
- Back-of-envelope sizing (the habit worth keeping): 100 M new links/month ≈ 40 writes/s avg; ×100 reads ≈ 4 k redirects/s avg, peaks 10×. 5 years ≈ 6 B rows × ~500 B ≈ 3 TB. That is a large single table, not "big data". Key space: **base62, 7 chars = 62⁷ ≈ 3.5 trillion**, which is far more than you need. 6 chars ≈ 57 B is often enough.

## Generating the short code (the real design decision)

| Approach | How | Trade-offs |
|---|---|---|
| **Counter + base62** | Unique integer ID → base62 encode | No collisions, shortest codes. But **sequential = enumerable**: anyone can walk `…aaB`, `…aaC` and scrape every link (privacy leak for "unlisted" docs). A single global counter is also a write bottleneck and a single point of failure. |
| **Range/block allocation** | Each app node leases a block of IDs (e.g. 1 M at a time) from a coordinator (DB sequence, ZooKeeper/etcd); hands them out locally | Removes the per-write coordination. A crash burns the rest of the block (gaps are fine). Still enumerable unless you scramble the integer first. |
| **Scrambled counter** | Unique ID → reversible bijection (e.g. Feistel network / format-preserving encryption, or a library such as Sqids with a shuffled alphabet) → base62 | Keeps collision-freedom and makes codes non-sequential. Sqids-style obfuscation is **not encryption**, so don't rely on it for secrecy. |
| **Random code** | Generate random 7–8 chars, insert with a **unique constraint**, retry on conflict | Simple and stateless. Collision probability grows with fill ratio (birthday bound), so keep the key space ≫ the row count. Retry must be **on the DB's unique-violation**, never "check then insert" (a race). |
| **Hash of the URL** (MD5/SHA → take first 7 chars) | Deterministic | Truncation **will** collide and needs the same insert-and-retry. Same URL → same code merges two users' links, which shares their analytics and their delete/expiry. Usually the wrong default, and adding a salt per user just turns it into the random approach. |

Default: **random or scrambled-counter codes, with a DB unique constraint as the single source of truth.** "Check if exists, then insert" is the classic race. Two writers both see "free" and both insert. Let the constraint arbitrate (see [[data-storage]]).

**Custom aliases** share the same namespace. Reserve system paths (`api`, `admin`, `login`, `static`, `health`). Normalise case consistently: base62 is case-sensitive, but humans retype codes in lowercase. Validate the charset so an alias can't contain `/`, `?`, `#` or `%`. A code is a path segment and must be escaped if you ever embed it in another URL.

**Idempotent creation:** a mobile client retrying a timed-out `POST` creates two links. Use an idempotency key, or return the existing code for (user, URL) if that is the product semantics (see [[api-design]]).

## The redirect: status codes are a product decision

- **301 / 308 (permanent)**: browsers and CDNs **cache the redirect indefinitely**. Repeat clicks never reach you, so **you lose analytics and can never change or disable the target** for that browser. Disabling a malicious link does nothing for clients that already cached it.
- **302 / 307 (temporary)**: every click hits your edge, so you can count it, change the target and kill it. This is what most commercial shorteners use. 307/308 preserve the method and body (matters only for non-GET).
- Send `Cache-Control` explicitly anyway. A 302 with `max-age` can still be cached. Use a short `max-age` at the CDN (`s-maxage=60`) and `private, max-age=0` for browsers if analytics or kill-switch matter more than origin load. This is the [[caching]] staleness budget applied to redirects: how fast must a disable take effect?
- **SEO:** a 301 passes link equity to the target and a 302 nominally doesn't (search engines are lenient in practice). Marketing teams sometimes demand 301 and then demand click stats. Say which they can have.
- Return 404 (or 410 for deleted/expired) for unknown codes. **Cache the 404 briefly (negative caching)**, otherwise scanners enumerating codes bypass every cache layer straight to the DB (cache penetration, see [[caching]]).

## Storage and the read path

- Data model is one key-value lookup: `code → {url, owner, createdAt, expiresAt, disabled}`. Any store works. Postgres with a primary key on `code` is fine for billions of rows. A KV store (DynamoDB/Cassandra) helps once you need multi-region active-active writes. Choose by operational fit, not by "it's a KV problem" (see [[data-storage]]).
- Read path: **CDN/edge → in-process LRU → Redis → DB**. The access distribution is extremely skewed (one viral link = most of the traffic). That makes the cache hit rate high, but it creates a **hot key**: put an L1 in front of Redis and let the CDN absorb it (see [[caching]]).
- Disabling or editing a link must **invalidate every layer**: CDN purge (surrogate key = code), Redis delete, L1 TTL. The slowest layer sets your real kill-switch latency.
- Expiry: check `expiresAt` on read (correctness), and delete expired rows asynchronously (housekeeping). Never depend on a cleanup job for correctness.
- Sharding, if ever needed: shard by `code` (hash). Every read is a single-key lookup, so there are no cross-shard queries on the hot path. "List my links" needs a secondary index by owner, and that is the one query that doesn't shard by `code`.

## Analytics without slowing the redirect

- Never write a click row synchronously in the redirect path. The redirect should do one cache read and return. **Emit a click event to a queue/log** (Kafka, Kinesis, or just structured access logs) and aggregate downstream (see [[async-systems]]).
- Counters: at-least-once delivery means duplicates, so count approximately (it's fine), or dedupe by event ID. Unique visitors → HyperLogLog, not a set per link.
- Bots (link-preview fetchers: Slack, iMessage, email security scanners) inflate clicks massively. Classify by user agent or behavior before reporting, or the numbers are meaningless. Email scanners also **pre-click** links, which matters if the redirect has side effects (one-time links, "confirm" URLs). Redirects must be safe GETs.

## Abuse is the main operational cost

- Shorteners are a **phishing and malware laundering tool**: the short domain hides the target. Expect it from day one. Check URLs against reputation services (e.g. Google Safe Browsing) at creation **and periodically after** (targets change after creation), allow user reports, and keep a fast kill switch. This is exactly why 302 matters.
- Rate-limit creation per account and IP, and require auth for bulk creation (see [[api-design]]).
- Block redirect loops and self-references (short URL → another short URL on your domain), and limit chains through other shorteners.
- Validate the target scheme: allow `http`/`https` only, never `javascript:`, `data:` or `file:`. If you render a preview/interstitial page, HTML-escape the target URL.
- Your domain's reputation is shared by all users. One spam wave gets the whole short domain blocklisted by mail providers and browsers, so **keep the shortener on its own domain**, separate from your main site and auth cookies.

## AEM instance

AEM does not need a shortener service for most "short URL" asks. The equivalents, and their limits:
- **Vanity paths** (`sling:vanityPath` on a page) give author-managed short paths on the same domain. They are unique, have no regex and must not collide with page paths. They need Dispatcher's `/vanity_urls` block to get past a strict `/filter`, and they can take ~5 min to propagate (`/delay 300`). See [url-resolution-layers-aemaacs](../dispatcher-url-delivery/url-resolution-layers-aemaacs.md). They can also **route around auth requirements** on protected trees, see [sling-resource-resolution-mapping](../dispatcher-url-delivery/sling-resource-resolution-mapping.md). They do not provide analytics, expiry or campaign codes on a separate domain.
- **Bulk campaign redirects** (marketing wants hundreds of `/promo-x` links) fit Apache `RewriteMap` files or the AEMaaCS CDN `redirects` rules better than vanity paths. However, both need a **pipeline deploy to change**. A redirect map in an Edge Functions **KV store** can change at runtime with no publish involvement, see [edge-functions](../cloud-service/edge-functions.md).
- A real shortener (separate short domain, click analytics, link expiry, user-created links) is a **separate service outside AEM**. Don't build it on the JCR: per-click writes on publish are the wrong workload. AEM just links to it.

## Design-review checklist

Code generation collision-free under concurrency (unique constraint arbitrates, no check-then-insert); codes non-enumerable if links can be private; 301-vs-302 decided against analytics + kill-switch needs, with explicit `Cache-Control`; disable/kill-switch latency measured across every cache layer; negative caching for unknown codes; click tracking async and bot-filtered; creation rate-limited and URLs reputation-checked at creation and after; target scheme allowlisted; short domain isolated from the main domain's cookies and reputation; redirect path survives with the analytics pipeline and creation API **down** (see [[reliability-patterns]]).
