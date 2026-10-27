# Dispatcher cacheability rules and the log line each one produces

Adobe's KCS article "Which requests does AEM Dispatcher cache?" (ka-17497, written for Dispatcher 4.3.3) is the one place that pairs every request-side and response-side cache rule with the **exact `dispatcher.log` message** it emits when the rule fails. That pairing is what makes it useful: a "why is this not cached?" question becomes a grep. Everything below is docs-derived, not reproduced against a running Dispatcher. The broader picture (invalidation, statfiles, flushing) is in [[dispatcher-caching-and-invalidation]]; other log lines are in [[dispatcher-common-log-entries]].

Adobe says up front that the article covers the most important rules, **not all of them**. Treat the lists as "if one of these fails it is not cached", not as "if all pass it is cached".

## Request rules (all must hold)

| # | Rule | Log message on failure | Not cached | Cached |
|---|---|---|---|---|
| 1 | URL is canonical: absolute, no `.`/`..` segments, **has a file extension**, and the filename maps to a path the filesystem accepts | `URI not canonical: %s.` · `Unable to map URI to file: %s.` · `cache file path too long` · `temporary file path too long` · `request URL has no extension` | `/content/test` | `/content/test.html` |
| 2 | **No slash after the first extension** (suffix) | `request URL has trailing slash` | `/content/test.html/` · `/content/test.html/foo.jpg` | `/content/test.html` |
| 3 | Method is **GET or HEAD** | `request method is neither GET nor HEAD` | `POST /content/test.html` | `HEAD /content/test.html` |
| 4 | `/cache` → `/rules` allows the path | `request URL not in cache rules` (older: `URI not in cache rules: %s`) | `/rules { /0001 { /type "deny" /glob "*" } }` | `/rules { /0001 { /type "allow" /glob "*" } }` |
| 5 | No `Authorization` **header**, or `/allowAuthorized "1"` | `request contains authorization` | `Authorization: Basic …` with `/allowAuthorized "0"` | same request with `/allowAuthorized "1"` |
| 6 | No `login-token` or `authorization` **cookie**, or `/allowAuthorized "1"` | `request contains authorization` (same line as rule 5) | `Cookie: login-token=…` or `Cookie: authorization=…` with `"0"` | same with `"1"` |
| 7 | Query string present only if every parameter is **ignorable** under `/ignoreUrlParams` | `request contains a query string` | `?test=1` with `deny *` + `allow q` | `?test=1` with `allow *`, or `deny *` + `allow test` |

Reading notes:

- **Rules 5 and 6 share one log line.** `request contains authorization` does not tell you whether it was the header or a cookie; look at the request. The three markers (header, `authorization` cookie, `login-token` cookie) are the complete list the Dispatcher checks, and the flag semantics ("may the security check be skipped?") plus the `/sessionmanagement` interaction are in [[dispatcher-permission-sensitive-caching]].
- **Rule 7 is "ignorable", not "absent".** With `/ignoreUrlParams` set to allow a parameter, the Dispatcher strips it from the cache key and caches the response under the bare URL, so `?utm_source=x` and the plain page share one file. Details and the cache-key consequences in [[dispatcher-ignoreurlparams]].
- **Rule 1's path-length lines** (`cache file path too long`, `temporary file path too long`) are the filesystem limit on long selector/suffix chains showing up as a cache miss rather than an error.

## Response rules (all must hold)

| # | Rule | Log message on failure | Not cached | Cached |
|---|---|---|---|---|
| 1 | A configured render returned **200 OK** | `Unable to send request to remote server.` · `Unable to receive response from remote server.` · `Remote server returned: %s` · `No backend available.` | `404 Page Not Found` | `200 OK` |
| 2 | None of `Dispatcher: no-cache`, `Cache-Control: no-cache`, `Pragma: no-cache` in the response | `Backend forbids caching: %s, sent: %s` · `response contains no_cache` | any of the three on a 200 | plain 200 |
| 3 | `Content-Length` **greater than zero** | `response content length is zero` | `Content-Length: 0` | `Content-Length: 3120` |

Reading notes:

- **Render `/timeout`.** The article's note on rule 1: if `/renders` has no `/timeout`, or it is `0`, the Dispatcher waits **forever** for a connection to a render that is down. Set it. The failure then shows as one of the rule-1 log lines instead of a hung request.
- **Only `no-cache` is named.** The article lists exactly three headers. The Dispatcher reference also treats `no-store` and `must-revalidate` as opt-outs ([[dispatcher-caching-and-invalidation]]); the KCS article simply does not mention them, so do not read its list as exhaustive.
- **`Dispatcher: no-cache`** is the AEM-side way to opt a single response out without touching browser or CDN caching: the Dispatcher honours it and does not forward it as a cache directive to clients the way `Cache-Control` would be.
- **Zero-length bodies** are never cached, which is why an empty but valid response (an endpoint that legitimately returns nothing) hits the render every time.

## Errors in the article itself

Two spots in the Adobe article are internally inconsistent; do not copy them into a config.

- Under rule 3 (GET/HEAD) the example table marks `GET /content/dam/test.jpg HTTP/1.1` as **not cached**. Nothing in the rule explains that. A GET for a DAM rendition with an extension is cacheable under every other rule in the article; treat that row as a copy-paste mistake.
- Under rule 7 the "not cached" example is headed `dispatcher.any farm /cache = /allowAuthorized configuration:` but shows an `/ignoreUrlParams` block, and both `/ignoreUrlParams` examples repeat the key `/0001` twice. In a real `dispatcher.any` the second `/0001` would collide; use `/0001`, `/0002`.

## Using it

When a URL is unexpectedly uncached, grep the log for the URL and match the line against the tables. In practice the order of discovery is usually: missing extension or trailing suffix (rule 1–2), a query string that is not ignorable (7), then an auth cookie you forgot the site sets (6), then a `no-cache` header the component or a filter added (response 2). With `/log/level` at debug the `cache-action` lines around the message tell you which farm handled it.

## What this means for AEM

On AEM 6.5/AMS all seven request rules and three response rules apply as written, and `/renders` → `/timeout` is yours to set. On AEMaaCS the same Dispatcher module runs inside Adobe's container with the same rules, but you author only the `cache/`, `filters/`, `clientheaders/` and related files of the dispatcher module; the farm-level `/renders` and `/timeout` are Adobe's. The log lines are the same, read through the Cloud Manager log download or the local `aemdispatcher` SDK.

Source: Adobe KCS article ka-17497, "Which requests does AEM Dispatcher cache?", Dispatcher 4.3.3. Docs-based.
