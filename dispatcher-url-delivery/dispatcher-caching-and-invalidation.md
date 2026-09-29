# Dispatcher caching, invalidation and `dispatcher.any` in one place

Condensed from Adobe's Dispatcher overview and the `dispatcher.any` reference,
plus a practitioner write-up on clearing the cache. Documentation-derived, not
reproduced against a running Dispatcher; version notes are Adobe's own. Related:
[[dispatcher-ignoreurlparams]] (query-string cache keying in depth),
[[sling-dynamic-include]] (keeping pages cacheable with dynamic fragments),
[[url-resolution-layers-aemaacs]] (where Dispatcher sits between CDN and Sling).

---

## What Dispatcher is

An Apache/IIS **module** in front of Publish that does two jobs: **cache** rendered
responses as files on disk, and **load-balance** across several renders. It is
not a proxy that understands AEM; it is a filesystem cache keyed by URL with a
timestamp-based invalidation trick.

- Cache files live under `/docroot` mirroring the URL path:
  `/content/site/en/page.html` → `<docroot>/content/site/en/page.html`.
- Only the **body** is cached by default. Response headers are cached only for
  the names listed in `/cache/headers` (4.1.11+), otherwise Apache re-derives
  them, so `Content-Type` for extension-less or odd URLs is lost.
- Docroot on **NAS** is explicitly warned against: slow, and shared NAS across
  Dispatchers produces intermittent locking during replication.
- OS filename-length limits bite URLs with many selectors.

## When a request is cacheable

Dispatcher **never** caches when any of these hold, regardless of `/rules`:

| Condition | Why |
|---|---|
| URI contains `?` | Query strings imply dynamic output (see [[dispatcher-ignoreurlparams]] for the exception) |
| No file extension | MIME type cannot be determined from the file name |
| `Authorization` header (or configured auth cookie) present | Unless `/allowAuthorized "1"` |
| Method is not GET or HEAD | POST etc. always go to the render |
| Response is not 200 | 50x are never cached, but see `/serveStaleOnError` |
| AEM answered `Cache-Control: no-cache`, `no-store` or `must-revalidate` | Render opted out |
| Zero-length body, or the request is for the statfile itself | |

Then `/cache/rules` decides by glob whether the URL is a cache candidate.

## The two invalidation mechanisms

**1. Content update (explicit delete).** On a flush request for `/en/index`,
Dispatcher deletes the cached file **and every sibling sharing the handle
prefix** (`/en/index.*`, so all selectors/extensions of that page), then
**touches the statfile**.

**2. Auto-invalidation (timestamp comparison).** Files matched by
`/cache/invalidate` are not deleted; on each request Dispatcher compares the
cached file's mtime with the relevant `.stat` file. Older than the `.stat` →
re-fetch from the render and overwrite. This is how *every* HTML page becomes
stale after *any* activation, which is what you want because pages embed
navigation and links to other pages.

Typical split: HTML is auto-invalidated (`/invalidate` allow `*.html`), assets
are not (`/invalidate` deny `*`), so an image stays cached until *its own*
handle is flushed.

### `.stat` files and `/statfileslevel`

- `/statfileslevel "0"` (default): one `.stat` in docroot; any activation
  invalidates every auto-invalidated file on the site.
- `/statfileslevel "N"`: `.stat` files are created in every directory down to
  depth N. On invalidation of a file, Dispatcher touches the `.stat` in that
  file's directory (or the deepest directory within N) **and every parent up
  to docroot**. Siblings' subtrees are untouched, so with level 3 an
  activation under `/content/site-a/en/` leaves `/content/site-b/en/` valid.
- A cached file is checked against the deepest `.stat` at or above it, so
  files sitting in shallow directories still expire on every activation.
- `/statfile` (single named file) is **ignored** when `/statfileslevel` is set.
- `/gracePeriod "2"`: keep serving auto-invalidated content for N seconds
  after the `.stat` touch. Throttles the thundering herd when a bulk
  activation touches `.stat` dozens of times a second.

### TTL and stale-on-error (4.1.11+)

- `/enableTTL "1"`: honour `Cache-Control: max-age` / `Expires` from the
  render; the file expires on its own. From 4.3.5 TTL and `.stat`
  invalidation both apply; before that TTL bypassed the statfile check.
- `/serveStaleOnError "1"`: if the render answers 502/503/504 and an
  invalidated copy exists, serve the stale copy instead of the error.

## Flushing the cache: the four ways

**Flush agent on Publish (Adobe's recommendation).** A "Dispatcher Flush"
replication agent on each Publish (`/etc/replication/agents.publish/flush`),
transport URI `http://<dispatcher>:80/dispatcher/invalidate.cache`, triggered
"on receive". Publish only flushes *after* it has received the content, so the
next request re-caches the new version. A flush agent on **Author** instead is
the classic race: Author flushes Dispatcher while Publish is still receiving
the replication, the next visitor re-caches the **old** page, and the site is
stale until the next activation. Author-side agents also need Author→Dispatcher
network reachability, which is usually not there.

**Manual HTTP invalidation.** Any client on `/cache/allowedClients` can POST:

```
POST /dispatcher/invalidate.cache HTTP/1.1
CQ-Action: Activate
CQ-Handle: /content/site/en/page
Content-Length: 0
```

`CQ-Action` is one of `Activate`, `Deactivate`, `Delete`, `Test`.
`CQ-Handle` is the content path without extension. Adding
`Content-Type: text/plain` and a body of one path per line makes Dispatcher
**re-fetch** those paths immediately after deleting them (delete + recache).
Lock `/allowedClients` to the Publish IPs; an open flush endpoint is a
trivial cache-busting DoS.

**Touch the `.stat`.** `touch <docroot>/.stat` (or a deeper one) invalidates
every auto-invalidated file under it without deleting anything. Cheapest
"everything is stale" switch; assets excluded from `/invalidate` are not
affected.

**Delete files.** `rm -rf <docroot>/content/site/*` while Apache runs is
safe; Dispatcher just misses and re-fetches. Needed after installing a hotfix
or feature pack that touches `/libs` or `/apps` clientlibs, since those files
are not auto-invalidated.

`/invalidateHandler "/path/script.sh"` runs a script on every invalidation
with handle, action and scope, the hook used for CDN purge integration.

## Load balancing between renders

- Dispatcher keeps per-render response-time **statistics** per
  `/statistics/categories` glob (max 8 categories, first match wins) and sends
  each request to the render with the best score for its category. Search
  pages and HTML pages therefore get separate averages.
- `/unavailablePenalty "1"` (tenths of a second) is added to a render's score
  when a connection fails, so it is de-prioritised, not removed.
- `/stickyConnectionsFor "/products"` or the multi-path `/stickyConnections`
  block pins a client to one render via a `renderid` cookie. Sticky paths are
  almost always **uncacheable** paths, otherwise the cache serves one user's
  personalised page to everyone.
- Retry: `/numberOfRetries "5"` rounds, `/retryDelay "1"` second between.
- `/failover "1"`: a 503 is resent to another render. Other 50x first hit
  `/health_check/url`; if the health page also fails the render is marked
  down, if it succeeds the original 500 is returned to the client.
- `/renders`: `/timeout "0"` (connect, wait forever), `/receiveTimeout
  "600000"` (10 min, then 504), `/secure "1"` for HTTPS, `/always-resolve "1"`
  to re-resolve DNS per request for dynamic backend IPs.

## `dispatcher.any` evaluation rules

- **Farms are evaluated bottom-up; virtualhosts inside a farm top-down.** Put
  the catch-all farm first so the specific ones below it win.
- `/filter`, `/rules`, `/invalidate`, `/allowedClients`, `/ignoreUrlParams`:
  **last matching rule wins**. The idiom is `/0001 deny *` first, then
  specific allows; the numbering is cosmetic, order is what matters.
- Filter on request-line elements (`/method /url /path /selectors /extension
  /suffix /query /protocol`), not on the deprecated `/glob` which matches the
  whole request line and is easy to bypass. Double quotes = glob, single
  quotes = regex (4.2.0+).
- `/clientheaders` is an allowlist; anything not listed never reaches the
  render. Forgetting `authorization`, `cookie` or `PATH` produces silent
  login and replication failures.
- `/sessionmanagement` requires `/allowAuthorized "0"` and a `/directory`
  that is not `/`.
- `/vanity_urls` polls `/libs/granite/dispatcher/content/vanityUrls.html`
  every `/delay` seconds and lets listed vanity URLs through **even when
  `/filter` denies them**.
- **Author Dispatcher must not cache**: `/rules { /0000 { /glob "*" /type
  "deny" } }`, otherwise Touch UI breaks.
- After changing `/invalidate` (and most of `/cache`) restart httpd; the
  section is read at startup.
- Debug: `/info "1"` plus request header `X-Dispatcher-Info` returns
  `X-Cache-Info`. `DispatcherLog` level 4 prints applied rules.

## AEMaaCS differences

Flush agents and `/dispatcher/invalidate.cache` wiring are preconfigured and
the filesystem is not reachable, so the `.stat`/`rm` options are gone; the
customer owns only the `dispatcher.any` fragments in the config repository,
validated by the SDK Dispatcher Tools. CDN purge is a separate operation from
Dispatcher invalidation. See [[url-resolution-layers-aemaacs]].

## Review checklist

- Site stale after every publish, fixed by a second activation: flush agent
  is on **Author**, move it to Publish.
- Images never update: they are excluded from `/invalidate`; flush the asset
  handle or add its extension to auto-invalidation.
- `Content-Type` wrong on cached responses: add it to `/cache/headers`.
- Whole site re-renders on any activation of a large multi-site instance:
  raise `/statfileslevel` to the site-root depth.
- Personalised content leaking between users: sticky path is cacheable.
- `/dispatcher/invalidate.cache` reachable from the internet: tighten
  `/allowedClients`.

## References
- [The Dispatcher overview (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-dispatcher/using/dispatcher)
- [Configuring Dispatcher, the dispatcher.any reference (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-dispatcher/using/configuring/dispatcher-configuration)
- [How to clear the Dispatcher cache (Axamit blog)](https://axamit.com/blog/adobe-experience-manager/dispatcher-clear-cache/)
