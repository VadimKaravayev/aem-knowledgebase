# Dispatcher fundamentals and farm configuration

What the Dispatcher *is* and the `dispatcher.any` sections the other notes
don't cover. Caching rules, invalidation, `/statfileslevel`, `/gracePeriod`,
`/allowedClients`, `/enableTTL` and flush mechanics live in
[[dispatcher-caching-and-invalidation]]; `/ignoreUrlParams` in
[[dispatcher-ignoreurlparams]]; `/allowAuthorized`, `/sessionmanagement` and
`/auth_checker` in [[dispatcher-permission-sensitive-caching]]; CORS header
interplay in [[cors-policy-and-dispatcher]].

---

## What the Dispatcher is

A **module inside an enterprise web server** (Apache httpd, IIS), not a
standalone process — the web server serves the cached files as ordinary static
content; the module only decides cache/forward/deny and talks to the renders.
Three jobs: **caching**, **load balancing**, **security** (the `/filter`
front door). **Dispatcher versions are independent of AEM versions** — you
update the module on the web server's cadence, not AEM's.

### How a request is answered

1. Is the document **cacheable** per the farm config? If not → forward to AEM.
2. Cacheable: does a cached file exist under `/docroot`? No → fetch from AEM,
   store, serve.
3. Exists: with auto-invalidation, compare the file's mtime against the
   relevant `.stat` — newer `.stat` means stale → re-fetch. Without
   auto-invalidation, **a cached document is served until it is physically
   deleted** — nothing ages out on its own (unless `/enableTTL` adds a clock).

Cached files mirror the URL path under `/docroot`, which is why the docroot
**must equal the web server's document root** (same files, two viewers) and
why **each farm needs its own docroot** — two farms sharing one directory
poison each other's cache.

### Two gotchas of file-based caching

- **Only the response body is stored.** Without a `/headers` section the
  cached hit carries whatever headers the *web server* synthesizes from the
  file (Content-Type guessed from the extension). Signature: first (MISS)
  response correct, cached hits with wrong/missing `Content-Type`,
  `Content-Disposition`, `Cache-Control`. Fix below under `/cache/headers`.
- **OS path-length limits apply.** A URL with many selectors can exceed the
  filesystem's filename limit — the request still works, it just silently
  never caches.

## `dispatcher.any` syntax

- Properties start with `/`, multi-values in `{ }`, comments with `#`.
- `/name` = instance label; `/farms` = the list of farms.
- **`$include "conf/farm_*.any"`** splits the config into files, wildcards
  allowed — how multi-site setups keep one farm per file.
- **`${VAR}`** interpolates environment variables into string values — the
  per-environment seam (e.g. `/hostname "${PUBLISH_HOST}"`).

## `/virtualhosts` — which farm answers

Entries are `[scheme]host[:port][/uri]` with `*` wildcards. Resolution walks
**farms bottom-up** and each farm's list **top-down**: a full match
(scheme + host + uri) beats a host-only match beats the fallback. Keep the
catch-all `*` virtualhost in the **topmost** farm so specific farms below it
win, and an unmatched request lands somewhere harmless.

## `/renders`

| Property | Default | Meaning |
|---|---|---|
| `/hostname`, `/port` | — | the AEM instance |
| `/timeout` | `"0"` = **wait forever** | connect timeout, ms — set it; the default hangs web-server threads on a dead render |
| `/receiveTimeout` | `"600000"` (10 min) | response timeout, ms |
| `/secure` | `"0"` | `"1"` = HTTPS to the render |
| `/ipv4` | `"0"` | `"1"` forces `gethostbyname` (IPv4-only resolution) |
| `/always-resolve` | `"0"` | `"1"` re-resolves DNS per request (renders behind changing IPs) — otherwise the first resolution is cached |

Multiple renders in one farm = load balancing (below).

## `/filter` — anatomy and the glob deprecation

Rules are numbered, **last match wins**, and the baseline must be deny-all
(`/0001 { /type "deny" /url "*" }`) with narrow allows after it.

- **Element-based rules (Dispatcher 4.2.0+) are the current form**: match on
  `/method`, `/url`, `/query`, `/protocol`, and the Sling-decomposed
  `/path`, `/selectors`, `/extension`, `/suffix`. Decomposed elements are the
  point: a deny on `/selectors '(feed|infinity|tidy)'` can't be dodged the way
  a URL-string glob can.
- **`/glob` matches the entire request line** (`GET /path?q=1 HTTP/1.1`) and
  is **deprecated** — it's the historically bypassable form (query-string and
  encoding tricks live in the same string being matched).
- Quoting selects the matcher: `"double quotes"` = glob pattern,
  `'single quotes'` = **regex** (4.2.0+), e.g. `/extension '(json|xml)'`.
- Adobe: **purge the cache after changing `/filter`** — already-cached files
  otherwise keep serving content the new rules would block.
- Post-change security test: `/admin`, `/crx`, `/system/console`,
  `/jcr:system/...`, `.infinity.json` etc. must all 404 through the
  Dispatcher. A valid page with `?debug=layout` should still render.

## `/cache` additions not in the caching note

- **`/headers` (4.1.11+)**: list of header *names* (no globs) stored in a
  sidecar file per cached document and replayed on hits. Default set:
  `Cache-Control`, `Content-Disposition`, `Content-Type`, `Expires`,
  `Last-Modified`, `X-Content-Type-Options`. To serve a cached `ETag`, add it
  here **and** set Apache `FileETag none` (or Apache stamps its own
  file-based ETag over it).
- **`/invalidateHandler`**: a script invoked per invalidation with
  `handle action actionscope` arguments — the hook for propagating a flush to
  an application-specific cache (custom CDN, search index, sibling store).
- **`/mode`**: octal permissions for created cache files/dirs, default
  `0755`, filtered through the process `umask` — the knob when a separate
  cleanup/rsync user can't read the cache.
- **`/serveStaleOnError "1"`**: on render connection failure / 502 / 503 /
  504, serve the already-invalidated cached copy instead of the error —
  availability stopgap while the publish farm is down.

## Load balancing internals

Per-farm `/statistics` `/categories` (glob-classified, **max 8, first 8
only**, most specific first) each keep a rolling response-time score per
render. Selection: honor the `renderid` **cookie** if present (sticky) →
otherwise the render with the **lowest score for the request's category** →
fallback: first render in the list. `/unavailablePenalty` (tenths of a
second, default `"1"`) is added to a render's score when TCP connect fails.

- **Retry loop**: `/numberOfRetries` (default `"5"`) rounds ×
  `/retryDelay` (default `"1"` s) between rounds; each round tries **every**
  render, so total attempts = retries × renders.
- **`/failover "1"`**: HTTP 503 → immediately try another render; other 50x →
  probe `/health_check` `/url` first — health check fails → next render,
  health check OK → the 500 was page-specific, return it to the client.
- **Sticky connections**: `/stickyConnectionsFor "/products"` (one folder) or
  `/stickyConnections { /paths { ... } }` (several trees composing one page).
  Cookie hardening on the `/stickyConnections` node: `httpOnly "1"` (default
  `"0"`), `secure "1"` = always set the secure flag (default `"0"` = only
  when the incoming request was HTTPS). Sticky paths must be uncacheable —
  a cached response never consults the cookie.

## Author Dispatcher

A Dispatcher in front of **Author** is legitimate (Adobe ships a separate
`author_dispatcher.any`) but: **do not cache author content with Touch UI**,
and after installing a feature pack/hotfix/service pack, **delete cached
`/libs` and `/apps`** or authors keep loading the pre-patch UI from cache.
(`/rules` with `deny *` remains the safe author-farm baseline — see
[[dispatcher-caching-and-invalidation]].)

## Logging and request debugging

- Log level in the web-server config (`DispatcherLogLevel` / module directive):
  **3 = Debug** for bring-up, **0 in production**, **4 = Trace (4.2.0+)**
  logs every forwarded header and *which filter rule blocked a request*
  (`'GET /content.infinity.json' was blocked because of /0082`) — the fastest
  answer to "why is this 404 through Dispatcher but fine direct".
- **`X-Dispatcher-Info`**: with `/info "1"` in the farm, a request sent with
  header `X-Dispatcher-Info: true` gets back `X-Cache-Info` naming the cache
  verdict and *why* (cached / caching / not cacheable: no extension, query
  string, authorization…) — per-request cache triage without log access.
- Bring-up checklist: loglevel 3 → start web server → `Dispatcher
  initialized (build …)` in the log → same page via AEM port and web-server
  port → cache files appear under docroot → activate a page and watch the
  flush → loglevel 0.

## Exam checklist

- Module in the web server; versioned **independently of AEM**.
- Never auto-expires without `.stat`-based invalidation or `/enableTTL`;
  body-only cache unless `/headers` is configured.
- One docroot per farm, equal to the web server document root.
- `/filter`: deny-all baseline, element rules over deprecated `/glob`,
  single quotes = regex, purge cache after changes.
- `/timeout` default is infinite — always set it.
- Load balancing: ≤ 8 statistics categories, `renderid` cookie wins, lowest
  category score next; failover 503 → next render, other 50x → health check.
- Author Dispatcher: no caching with Touch UI; flush `/libs` + `/apps` after
  patches.

## References
- [Overview of the Dispatcher (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-dispatcher/using/dispatcher)
- [Configuring Dispatcher (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-dispatcher/using/configuring/dispatcher-configuration)

Docs-based (both pages read Oct 2026), not lab-verified.
