# API design — REST, pagination, idempotency, versioning, rate limiting

General system-design reference (not AEM-specific). Key facts, trade-offs, and the design traps interviewers and incident reviews both love.

## REST semantics that actually matter

- Methods by contract, not convention: **GET/HEAD = safe** (no state change), **PUT/DELETE = idempotent** (N calls ≡ 1 call), **POST = neither** by default, **PATCH = partial update, not idempotent unless you make it so** (JSON Merge Patch applied twice is idempotent; "increment field" PATCH is not).
- Status codes carry retry semantics: `4xx` = caller's fault, **don't retry unchanged**; `5xx`/`429` = retryable (with backoff); `409` = state conflict (re-read, then decide); `422` = understood but invalid. Returning `200` with an error body breaks every generic client, retrier, and monitor downstream.
- `PUT` to a client-chosen ID is the cheapest idempotent create. `POST /things` returning `201` + `Location` is the server-chosen-ID create.
- Resource naming: nouns + hierarchy (`/orders/42/items`), actions that don't map to CRUD become sub-resources (`POST /orders/42/cancellation`) rather than RPC verbs in the path.

## Pagination

- **Offset pagination** (`?offset=100&limit=20`): simple, supports "jump to page N", but (1) **O(offset) cost** on most stores — the DB walks and discards the skipped rows, page 10,000 is a full scan; (2) **drift under writes** — a row inserted/deleted while paging shifts the window → duplicates or missed rows.
- **Cursor/keyset pagination** (`?after=<opaque cursor>`): cursor encodes the last-seen sort key (`WHERE (created_at, id) > (:ts, :id) ORDER BY ... LIMIT n`); O(1) per page with a matching index, stable under concurrent inserts. Cost: no random page access, and the sort key must be **unique and immutable** (tie-break composite keys with `id`, never paginate on a mutable column alone).
- Make cursors **opaque** (base64 of the key material) — clients that parse them pin you to the implementation forever.
- Always return a `next` cursor/link even when you think it's the last page; "absence of next = done" is the contract.
- Trap: `COUNT(*)` totals on every page are often more expensive than the page itself — make totals optional or approximate.

## Idempotency

- The problem: **network ambiguity** — a timeout tells the caller nothing about whether the server executed the request. The caller WILL retry; design for the duplicate, don't hope it away.
- **Idempotency keys** (Stripe pattern): client sends `Idempotency-Key: <uuid>` on POST; server atomically records key → stores the first response → replays that stored response for duplicates. Key points: the key+response record must be written **in the same transaction** as the side effect (or via a unique constraint on the key), keys get a TTL (24h typical), and a retry with the same key but a **different body** should be rejected (`422`), not silently replayed.
- Idempotency at the API layer doesn't compose automatically with downstream calls — if handling the request calls two non-idempotent services, a crash between them still half-executes. That's where sagas/outbox come in — see [[async-systems]].
- Natural idempotency beats bookkeeping where available: `PUT` full state, "set status to X" instead of "toggle status", unique business keys (`order_number`) with upsert.

## Versioning

- Options: URL path (`/v2/...` — visible, cache-friendly, easy routing), header/media-type (`Accept: application/vnd.x.v2+json` — "pure" but opaque to logs, caches, and humans), query param (worst of both). Path versioning wins in practice; the debate is a bikeshed.
- The real discipline is **what counts as breaking**: removing/renaming a field, changing a type, tightening validation, changing error shapes = breaking. **Adding optional fields is not** — therefore clients must ignore unknown fields (tolerant reader). Enforce this in client codegen settings (e.g. don't fail on unknown enum values — add an `UNKNOWN` fallback; a new enum member otherwise breaks every old client, the classic silent break).
- Version the **surface, not each resource**: per-endpoint version matrices are unmanageable.
- Plan deprecation as part of versioning: `Deprecation`/`Sunset` headers, usage metrics per version per client, and a date — "v1 forever because we can't see who uses it" is the default failure mode (so: see per-client telemetry in [[observability]]).

## Rate limiting

- Algorithms: **fixed window** (simple, but 2× burst at window boundary), **sliding window log/counter** (accurate, more state), **token bucket** (rate + burst capacity, the usual right answer), **leaky bucket** (smooths output, queues internally).
- Return **`429` + `Retry-After`**, and advertise state with `RateLimit-Limit/Remaining/Reset` headers — clients can't behave well blind.
- Decide the **key**: per API key / per user / per IP / per tenant. Per-IP breaks behind corporate NAT (many users, one IP — see [ip-addresses](../network/ip-addresses.md)); per-tenant protects the platform from one noisy tenant (the actual goal in multi-tenant systems).
- Distributed limiters need shared state (Redis `INCR`+`EXPIRE`, or a Lua-scripted token bucket for atomicity); local-only counters multiply the real limit by the node count. Accept approximation — a perfectly consistent global limiter is slower than the thing it protects.
- Rate limiting is **load shedding at the edge**: reject early and cheap rather than letting excess load reach the expensive tier — the same philosophy as backpressure in [[reliability-patterns]].
- Trap: retries without jitter turn a rate-limited burst into a synchronized retry storm; the limiter and the retry policy must be designed together.

## Cross-cutting gotchas

- Any value interpolated into a URL/query string must be encoded for that syntax first — unescaped delimiters fail silently (output degrades rather than erroring).
- Timeouts, retries and limits are part of the API contract — publish them (max page size, rate limits, timeout budget), or every client discovers them in production.
- AEM tie-in: Adobe's Cloud Manager API is a worked example of hypermedia design (HAL links, navigate-by-rel, never hand-build URLs) — see [cloud-manager-api-cli-sdks](../cloud-service/cloud-manager-api-cli-sdks.md); IMS token caching (~24h tokens, cache or get throttled) is rate limiting meeting auth — see [adobe-ims-authentication-types](../cloud-service/adobe-ims-authentication-types.md).
