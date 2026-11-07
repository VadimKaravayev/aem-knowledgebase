# Caching GraphQL persisted queries in the AEMaaCS dispatcher, and the CORS vhost that comes with it

Docs-based: Experience League, AEMaaCS headless deployment page "GraphQL Persisted Queries – enabling caching in the Dispatcher". Analysis of the example vhost (regex weakness, header precedence) is my own reading of the directives, flagged where it is inference. How this relates to the OSGi CORS policy is in [[aemaacs-headless-cors]].

## Off by default, and why

Persisted-query caching in the dispatcher is **disabled by default**. Adobe's reason: customers using CORS with **multiple origins** must first review, and possibly update, their dispatcher configuration. The dispatcher **does not cache the `Vary` header**. Other CORS headers can be cached, but a cached `Access-Control-Allow-Origin` holds one origin, which is wrong for every other origin. So switching caching on is really a CORS migration.

## Enabling it

1. Add **`Define CACHE_GRAPHQL_PERSISTED_QUERIES`** to the dispatcher's `global.vars`.
2. A rewrite rule (Adobe's, activated by the flag) appends a **`.json` suffix** to persisted-query URLs so the dispatcher can store them as files. You do not write this rule.
3. Add **`FileETag Digest`** inside the vhost's `<Directory />` block if it is not there. Each cached response then gets a content-based ETag instead of one derived from file metadata.
4. Move CORS to the dispatcher (next section) and **stop forwarding `Origin` to publish**: check `clientheaders.any`. Once caching is on, Adobe says the AEM CORS filter is not needed.

The page does not cover TTL / `Cache-Control`, invalidation, or error-response caching.

## The CORS vhost, decoded

Adobe's example classifies every request with `SetEnvIfExpr`. Later matches override earlier ones:

| Condition | `CORSType` | Processing |
|---|---|---|
| no `Origin` | `none` | false |
| `Origin` present | `cors` | true, untrusted until proven |
| `OPTIONS` + `Origin`, no `Access-Control-Request-Method` | `invalidpreflight` | false |
| `OPTIONS` + `Origin` + `Access-Control-Request-Method` | `preflight` | true, untrusted |
| `Origin` matches `https://%{HTTP_HOST}*` | `samedomain` | false |

Then:

- **Trust test:** `env('CORSProcessing') == 'true' && req_novary('Origin') =~ m#(https://.*.your-domain.tld(:\d+)?$)#` sets `CORSTrusted=true`. This is the line you adapt.
- `SetEnvIfNoCase ^Origin$ ^https://(.*)$ CORSTrustedOrigin=https://$1` captures **any** https origin. It is only emitted when `CORSTrusted` is true.
- **Trusted:** `Header always set` gives `Access-Control-Allow-Origin: <request origin>`, `Access-Control-Allow-Credentials: true`, `Access-Control-Allow-Methods: GET`, `Access-Control-Max-Age: 1800`, and `Access-Control-Allow-Headers: Origin, Accept, X-Requested-With, Content-Type, Access-Control-Request-Method, Access-Control-Request-Headers`.
- **Not processed or untrusted:** the same headers are `Header unset`.
- **Always:** `Header merge Vary Origin` (for browsers and the CDN; the dispatcher itself ignores it).
- **Untrusted CORS request:** `RewriteCond expr "reqenv('CORSProcessing') == 'true' && reqenv('CORSTrusted') == 'false'"` + `RewriteRule "^(.*)" - [R=204,L]`. The answer is an empty **204**, not a 403. The browser then fails on the missing `Access-Control-Allow-Origin`, so expect a CORS error in the console, not an HTTP error.

Because the headers are computed **per request** at the dispatcher, after the cache, one cached file can serve every trusted origin. That is the multi-origin problem from [[cors-policy-and-dispatcher]] solved.

## Gotchas

- **Copying the trust regex as-is is an origin bypass** (inference from regex semantics). In `https://.*.your-domain.tld` the dots are unescaped and `.*` is greedy. With `your-domain.tld` replaced by `example.com`, the pattern also matches `https://evilexample.com`, a domain anyone can register. Combined with `Allow-Credentials: true`, that origin can make credentialed reads. Write it as `^https://([a-z0-9-]+\.)*example\.com(:\d+)?$`.
- **GET only.** `Access-Control-Allow-Methods` is `GET`, which suits persisted queries. Ad-hoc POSTs to the GraphQL endpoint are not covered here. They need the OSGi CORS policy and a Referrer Filter entry ([[sling-referrer-filter]]).
- **The dispatcher overrides AEM** (inference). `Header always set` / `Header unset` replace whatever CORS headers publish returned. With this vhost, an OSGi CORS policy for the same paths has no visible effect, and since `Origin` is no longer forwarded it would not match anyway.
- **Do not mix models.** Forwarding `Origin` (needed by the OSGi policy) while also caching persisted queries brings back one-origin-per-cached-file.
- **`Access-Control-Allow-Credentials: true` is unconditional for trusted origins**, unlike the OSGi example (`supportscredentials: false`). That is harmless for anonymous persisted queries, but it is one more reason the trust regex must be tight.
- **Same-domain detection** uses `-strcmatch 'https://%{HTTP_HOST}*'`, a glob prefix match. Behind a CDN, `HTTP_HOST` is whatever host the CDN forwards, so verify it matches the public domain, or same-domain requests are processed as CORS.

## What this means for AEM

- **AEMaaCS:** this is the recommended delivery path for headless. Persisted queries over GET, cached at the dispatcher and the CDN, with CORS in the vhost. Configuration ships in the dispatcher module through Cloud Manager.
- **6.5 / AMS:** the `CACHE_GRAPHQL_PERSISTED_QUERIES` flag is part of the AEMaaCS dispatcher SDK layout. The vhost CORS technique can be ported to an AMS dispatcher by hand, along with a rule that makes persisted-query URLs cacheable.

Related: [[aemaacs-headless-cors]], [[cors-policy-and-dispatcher]], [[sling-referrer-filter]], [[aemaacs-caching-cdn-dispatcher-browser]].

Source: Adobe Experience League, "GraphQL Persisted Queries – enabling caching in the Dispatcher" (AEMaaCS headless deployment).
