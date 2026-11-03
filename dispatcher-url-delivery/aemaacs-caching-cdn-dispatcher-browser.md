# AEMaaCS caching: CDN, Dispatcher and browser TTLs, blob redirects, invalidation, clientlib versioning

What Adobe's "Caching in AEM as a Cloud Service" page actually commits to, reorganised around the decisions you make: which layer caches what by default, how to override it per path, what the CDN refuses to cache, how invalidation reaches the Dispatcher when you cannot touch it, and why clientlib URLs survive deployments. Docs-based (Adobe Experience League, AEMaaCS content-delivery section); not lab-verified.

## The request path and who sets the TTL

```
browser ── Adobe CDN (Fastly) ── Apache + Dispatcher ── AEM Publish ── blob store
```

- The CDN caches on origin response headers only: `Cache-Control`, `Surrogate-Control`, or `Expires`. There is no CDN-side TTL table to edit; you set headers in the Dispatcher vhost with `mod_headers`, or from Java on Publish.
- `Surrogate-Control` is read by the CDN and ignored by browsers, so it is the lever for "long at the edge, short in the browser". It applies to the Adobe-managed CDN; a customer CDN in front may need its own header.
- `Age` is set to `0` in every Adobe example so the CDN starts its TTL clock at the response, not at Dispatcher's cache time. The only reason to disable defaults (`Define DISABLE_DEFAULT_CACHING` in `global.vars`) is business logic that needs a computed `Age`.
- Dispatcher honours the same expiry when `enableTTL` is on: it re-fetches expired files even without a republish.

## What the CDN will never cache

- `Cache-Control` containing `private`, `no-cache` or `no-store`.
- Any response carrying `Set-Cookie`.
- The cache key is the **full URL including query string**, so each distinct parameter set is a separate entry (see marketing parameters below).
- HEAD on a miss is converted to GET toward Dispatcher/AEM. If the response is cacheable, later HEADs are CDN hits; if not, HEADs keep going to origin for the `Cache-Control` TTL.

## Defaults by content type

| Content | Default TTL | Where the default comes from | Override |
|---|---|---|---|
| `text/html` | 5 min browser, CDN follows | Dispatcher vhost | `Define EXPIRATION_TIME` in `global.vars` (all HTML) or a `LocationMatch` (per path) |
| Clientlib JS/CSS | `immutable`, 30 days for browsers that ignore `immutable` | Clientlib framework (hash in URL) | Not needed; see version consistency |
| Blob-stored content (images, large binaries) | Program id > 65000: `public, max-age=600, immutable` (anonymous) / `private, max-age=600, immutable` (authenticated). Program id <= 65000: **no caching** | AEM, only when no `Cache-Control` was already set | `LocationMatch`, or env var `AEM_BLOB_ENABLE_CACHING_HEADERS` (`true` enables on old programs, `false` reverts new ones, not recommended) |
| Other node-store types (JSON, txt, ...) | **None** | - | `LocationMatch` only; `EXPIRATION_TIME` does not apply |

The program-id threshold corresponds to programs created after mid-May 2022.

### Per-path override pattern

```apache
<LocationMatch "^/content/.*\.(html)$">
     Header set Cache-Control "max-age=200"
     Header set Surrogate-Control "max-age=3600"
     Header set Age 0
</LocationMatch>
```

AEM strips caching headers from responses it considers uncacheable. To force them anyway, unset first and use `always`:

```apache
<LocationMatch "^/content/.*\.(html)$">
     Header unset Cache-Control
     Header unset Expires
     Header always set Cache-Control "max-age=200"
     Header set Age 0
</LocationMatch>
```

Prerequisite in `src/conf.dispatcher.d/cache`: a rule `/0000 { /glob "*" /type "allow" }` (present in the archetype default).

### Making a path private at the CDN

```apache
<LocationMatch "/content/secure/.*\.(html)$">
   Header unset Cache-Control
   Header unset Expires
   Header always set Cache-Control "private"
</LocationMatch>
```

Two consequences the page spells out:

- `private` keeps the CDN out, but Dispatcher may still cache by its own rules. Truly private content must also be excluded from the Dispatcher cache, or guarded with permission-sensitive caching ([[dispatcher-permission-sensitive-caching]]).
- Private **blob** content (images) cannot be covered by permission-sensitive caching at all; it always goes to origin and is served only to authorised users.

Adobe's note: other override mechanisms, "including the dispatcher-ttl ACS Commons project", do not successfully override these values.

## Blob redirects (the 302 you did not write)

JCR binaries larger than 16 KB live in blob storage and AEM serves them as **302 redirects**. The CDN follows the redirect and streams the binary from the blob store directly. Only a short list of response headers can be customised on these redirected responses: `cache-control`, `vary`, `content-disposition`, `x-robots-tag`, `referrer-policy`, `content-security-policy`, `x-frame-options`, `x-xss-protection`, `x-content-type-options`, `access-control-allow-origin`, `permissions-policy`, `x-vhost`. Anything else set in the vhost is dropped. Example from the page, forcing inline PDFs:

```apache
<LocationMatch "\.(?i:pdf)$">
  ForceType application/pdf
  Header set Content-Disposition inline
</LocationMatch>
```

Design implication: a Dispatcher `/filter` or header rule that assumes it sees the binary bytes is wrong for AEMaaCS; the bytes never pass through Apache.

## Adobe's recommended header set

All rules carry `"expr=%{REQUEST_STATUS} < 400"` so error responses are not cached, and `Header set Age 0`. Adobe suggests a 30-minute `stale-while-revalidate` on everything as a starting point; these examples use 12 hours (43200 s).

| Target | Header |
|---|---|
| `/etc.clientlibs/` mutable (json, png, gif, webp, jpg, svg) | `Cache-Control "max-age=43200,stale-while-revalidate=43200,stale-if-error=43200,public"` |
| `/etc.clientlibs/` immutable (js, css, ttf, woff2) | `Cache-Control "max-age=2592000,stale-while-revalidate=43200,stale-if-error=43200,public,immutable"` |
| `/content/*.html` | `Header always set Cache-Control "max-age=300,stale-while-revalidate=3600"` + `Header always set Surrogate-Control "stale-while-revalidate=43200,stale-if-error=43200"` |
| `/content/*.model.json` | same values as HTML, with plain `Header set` |
| `/content/*.coreimg.*.(jpg,png,gif,svg)` | `Cache-Control "max-age=2592000,stale-while-revalidate=43200,stale-if-error=43200,public,immutable"` |
| `/content/dam/*` (jpg, gif, js, mov, mp4, png, svg, txt, zip, ico, webp, pdf) | `Cache-Control "max-age=43200,stale-while-revalidate=43200,stale-if-error=43200"` |

Reading the HTML row: 5 minutes fresh in the browser, 1 hour of background refresh in the browser, 12 hours of stale-while-revalidate and stale-if-error at the CDN. The `always` on HTML means the header lands on everything the regex matches, so the regex must match only public pages.

Also from the optimisation list: remove `Header append Vary User-Agent env=!dont-vary` from every vhost except the read-only `default.vhost` (archetype versions before 28 shipped it). `Vary: User-Agent` fragments the CDN cache per browser string.

## Marketing query parameters

Two layers cooperate:

1. **CDN** (environments created October 2023 or later) strips parameters matching `^(utm_.*|gclid|gdftrk|_ga|mc_.*|trk_.*|dm_i|_ke|sc_.*|fbclid|msclkid|ttclid)$` before the cache key is computed. Toggle via CDN config:

   ```yaml
   kind: "CDN"
   version: "1"
   data:
     requestTransformations:
       removeMarketingParams: false
   ```

2. **Dispatcher** `/ignoreUrlParams`, still recommended even with the CDN stripping. Preferred form is deny-by-default, because it also neutralises cache-busting parameters:

   ```
   /ignoreUrlParams {
      /0001 { /glob "*" /type "allow" }
      /0002 { /glob "page" /type "deny" }
      /0003 { /glob "product" /type "deny" }
   }
   ```

   Remember the inverted vocabulary: `allow` = ignore the parameter, `deny` = it is cache-relevant and passed to Publish ([[dispatcher-ignoreurlparams]]). The alternative, `/glob "*" /type "deny"` plus `$include "../cache/marketing_query_parameters.any"`, uses an archetype list that Adobe does **not** maintain.

## Invalidation on a Dispatcher you cannot log into

Default stance: do not invalidate manually. Publishing and unpublishing flush the affected path and its parents up to `statfileslevel`; the CDN simply waits for TTLs (Dispatcher invalidation does **not** purge the CDN).

Flush requests arrive from Publish with Host values `127.0.0.1`, `localhost`, `*.local`, `*.adobeaemcloud.com`, `*.adobeaemcloud.net`. A vhost must match them or flushes are silently dropped; the archetype's catch-all `ServerAlias *` vhost is the usual answer. "Pages never refresh after publish" on a project that trimmed the vhosts is this.

When explicit invalidation is justified (Experience Fragments or Content Fragments referenced by many pages, or notifying an external system):

| | Sling Content Distribution (preferred) | Replication API |
|---|---|---|
| Triggered from | Author | Publish (flush agent) |
| Guarantee | At least once | Best effort |
| Deduplication | Discovery API or dedup mode | Not possible, event fires on every Publish |
| Cache action | `INVALIDATE` = invalidate without publishing (resource-only) | `DEACTIVATE`/`DELETE` on the flush agent = invalidate only |

SCD call from Author:

```java
DistributionRequest req = new SimpleDistributionRequest(DistributionRequestType.INVALIDATE, false, "/content/to/invalidate");
distributor.distribute("publish", resolver, req);
```

On Publish, react via an `EventHandler` on `AGENT_PACKAGE_DISTRIBUTED`, check `DISTRIBUTION_TYPE == INVALIDATE`, and run only on the topology leader (`discoveryService.getTopology().getLocalInstance().isLeader()`) to avoid one action per Publish pod. The Replication API variant filters agents by id `flush` and calls `Replicator.replicate(session, DELETE, paths, options)`; Adobe notes it mainly fits triggers originating on Publish, or setups using Dispatcher TTL. Background on the journal model in [[content-distribution-journal]].

## Clientlib version consistency

- The HTML Library Manager appends a content hash to clientlib URLs (`clientlib-base.lc-<hash>-lc.css`). The hash changes only when the library's CSS/JS changes, so unrelated deployments leave browser caches intact.
- Strict versioning with long-term cache key `/.*;hash` is **enabled automatically** on dev, stage and prod. On the local SDK you tick "Strict Versioning" in the Adobe Granite HTML Library Manager config yourself; do not commit it.
- **Short-cache fallback**: cached HTML can reference an `lc-` URL that a deployment removed. AEM then redirects to an `sc-` URL carrying a timestamp segment (`clientlib-base.sc-<hash>-<epoch>-sc.css`) that resolves to the current version. Redirect and response both use `max-age=60`; downstream caches must not lengthen it or treat `sc-` URLs as immutable.
- Consequence: clientlibs must stay backward compatible with any HTML still in cache. Long HTML TTLs widen that window.

## Design checklist

- Decide browser vs CDN TTL separately: `Cache-Control` for the browser, `Surrogate-Control` for the edge.
- Guard every `Header set` with `expr=%{REQUEST_STATUS} < 400`.
- Secured content needs **both** `Cache-Control: private` (CDN) and a Dispatcher rule or permission-sensitive caching (Dispatcher).
- Binaries over 16 KB are redirects; only the listed headers survive.
- Keep the catch-all vhost; flushes depend on it.
- Prefer SCD `INVALIDATE` from Author over Replication-API flushes from Publish.
- Never extend TTLs on `sc-` clientlib URLs.

## Related

- Classic Dispatcher cache rules and `.stat` mechanics: [[dispatcher-caching-and-invalidation]], [[dispatcher-cacheability-rules-and-log-messages]].
- Which layer owns which URL rule on AEMaaCS: [[url-resolution-layers-aemaacs]].
- Content Request counting at the CDN ingress, which these TTLs directly affect: [[aemaacs-licensing-content-requests]].
- Generic `Cache-Control` semantics: [[caching]].
