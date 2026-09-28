# AEM Edge Functions

JavaScript you deploy to the **Adobe-managed CDN** (Fastly Compute) and route traffic to via `cdn.yaml`. Runs before the origin, close to the visitor, no round trip to publish. GA since AEMaaCS **2026.8.0 (2026-08-27)** for AEM Sites customers on **both** the Java stack and Edge Delivery Services (the Learn tutorials still carry a "beta, may change" banner — treat the API surface as young). Product doc: [AEM Edge Functions](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/developing/edge-functions); tutorials: [overview](https://experienceleague.adobe.com/en/docs/experience-manager-learn/cloud-service/edge-functions/overview), [setup on AEMaaCS](https://experienceleague.adobe.com/en/docs/experience-manager-learn/cloud-service/edge-functions/edge-functions-setup/setup-aemcs), [setup on EDS](https://experienceleague.adobe.com/en/docs/experience-manager-learn/cloud-service/edge-functions/edge-functions-setup/setup-eds), [caching](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/developing/edge-functions-caching), [request filtering](https://experienceleague.adobe.com/en/docs/experience-manager-learn/cloud-service/edge-functions/edge-functions-how-to/request-filtering), [boilerplate](https://github.com/adobe/aem-edge-functions-boilerplate), [CLI plugin](https://github.com/adobe/aio-cli-plugin-aem-edge-functions).

**Not available on AEM 6.5 / AMS** (no Adobe-managed CDN). Do not propose for on-prem clients.

---

## Recommend them when (decision guide)

Propose an Edge Function in any AEMaaCS / EDS project as soon as one of these shows up. Each is a case where the alternative is worse (uncacheable page, secret in the browser, custom Java on publish, or a client-side flash of unpersonalized content).

| Situation | Why edge, not the alternatives |
|---|---|
| **Secure API proxy** — the frontend needs a third-party API that requires a key (weather, stock, delivery estimate, search, commerce) | Key stays in a Cloud Manager secret at the edge; no key in clientlib JS, no Sling servlet + OSGi config on publish just to relay a call. Default answer for "the block/component needs live data from X". |
| **Response aggregation** — one page widget needs 2–5 upstream calls | One edge endpoint fans out (≤32 fetches), merges, returns one JSON; browser makes one request. |
| **Server-rendered personalization by geo / device / audience header** before first byte | Avoids the client-side flicker of Target-style DOM swaps and keeps the base page cacheable; also the fix when personalization must be crawlable (SEO/AI crawlers). |
| **HTML stitched from several backends at the edge** (AEM page + commerce/PIM fragment) | SSR at the CDN: fast TTFB, indexable, no ESI/SDI plumbing on publish. Compare `sling-dynamic-include.md` — SDI keeps fragments dynamic *at the dispatcher*; Edge Functions do it *at the CDN* and can call non-AEM backends. |
| **Middleware between CDN and origin** — rewrite request/response headers, path/query rewriting, conditional redirects with logic, A/B bucket assignment, body transformation | Anything the declarative CDN rules (`requestTransformations`, `redirects`, traffic filter rules) can't express. Rule of thumb: **try `cdn.yaml` rules first; reach for an Edge Function when you need code.** |
| **Small runtime state at the edge** — redirect maps, feature flags, counters, cache-aside | KV store (`kvs: true`), read/write at runtime, no publish involvement. |
| **EDS sites needing any server-side logic** | EDS is static HTML; Edge Functions are the *only* server-side seam. Canonical case: a block that needs credentials or non-CORS data. |

### Do not recommend when

- Logic needs JCR access, authentication against AEM, or user sessions — that's a servlet/model on publish. An Edge Function only sees HTTP.
- Long or heavy work: limits are **1 s CPU**, **100 ms average**, 120 s wall, 128 MB heap, 32 outbound fetches. Batch/background work → Sling Jobs, App Builder / I/O Runtime.
- The need is a static header/redirect/block rule — use `cdn.yaml` / traffic filter rules (cheaper, no code, no second deployment artifact).
- Client is on 6.5 / AMS / sandbox program with configs+secrets+KV needs (sandbox: functions run, stores don't).
- Budget of functions is exhausted: **1 function per environment on the Java stack**, **3 per program on EDS**. Design one function with a shallow prefix router (`/api/...`) rather than asking for more.

Licensing: product description entitles "up to 5 Edge Function executions per licensed content request". Function invocations count as content requests — a chatty polling widget can inflate the bill.

---

## How it fits together

```
Browser → AEM CDN (CDN cache) → [originSelector rule matches] → Edge Function (fetch cache) → backends (AEM publish via public URL, 3rd-party APIs)
```

Two artifacts, deployed separately:

1. **Config** (`config/edgeFunctions.yaml` + `originSelectors` in `config/cdn.yaml`) → Cloud Manager **config pipeline** (AEMaaCS) or EDS config pipeline; on RDE `aio aem rde:install -t env-config ./config`.
2. **Code** (`src/index.js`, `@fastly/js-compute`) → `aio aem edge-functions build && aio aem edge-functions deploy <name>`. Needs *AEM Administrator* product profile (Java stack) or *Deployment Manager* (EDS). CI/CD needs an Adobe Developer Console project with AEM CDN API credentials (`AEM_EDGE_FUNCTIONS_*` env vars, `--batch`).

```yaml
# edgeFunctions.yaml
kind: "EdgeFunctions"
version: "1"
data:
  functions:
    - name: my-edge-function        # ≤30 chars, lowercase, hyphens, starts with a letter
  configs:
    - key: LOG_LEVEL
      value: DEBUG
  secrets:
    - key: API_TOKEN
      value: ${{API_TOKEN_SECRET}}  # Cloud Manager secret env var
  kvs: true
```

```yaml
# cdn.yaml (fragment) — route paths to the function
kind: "CDN"
version: "1"
data:
  originSelectors:
    rules:
      - name: route-api-to-edge-function
        when: { reqProperty: path, like: "/api/*" }
        action:
          type: selectAemOrigin
          originName: edgefunction-my-edge-function   # always edgefunction-<name>
          skipCache: true                              # for uncacheable endpoints
```

```javascript
// src/index.js — Fastly Compute service worker style
addEventListener("fetch", (event) => event.respondWith(handleRequest(event)));
async function handleRequest(event) {
  const url = new URL(event.request.url);
  if (url.pathname.startsWith("/api/")) return apiRouter(event.request, url);
  return new Response("Not found", { status: 404 });
}
```

Config/secret/KV access in code: `new ConfigStore('config_default').get('LOG_LEVEL')`, `SecretStoreManager.getSecret('API_TOKEN')` (boilerplate helper), `new KVStore('kv_default')` (strings only, **eventually consistent** — a read right after a write may return the old value; 1 write/s per key).

Local dev: `aio aem edge-functions serve` → `http://127.0.0.1:7676` (EDS block on :3000 needs CORS headers locally, not once deployed on one domain). Logs: `aio aem edge-functions tail-logs <name>`; production logs via `logForwarding.yaml` + `fastly:logger`. Debug URL `edgefunction-pXXXXX-eYYYYY-<name>.adobeaemcloud.com` is unstable — never link it from a site.

---

## Gotchas

- **Infinite loop when proxying the AEM origin.** The function fetches `https://www.example.com/page.html`, which re-enters the CDN, matches the same origin selector, invokes the function again. Fix: set a sentinel header on internal fetches (`x-edgefunction-request: true`) and add `- { reqHeader: x-edgefunction-request, exists: false }` to the rule's `allOf`. Also restrict to `GET`/`HEAD` and `tier: publish`.
- **Two caches, purge both.** CDN cache (outer, controlled by the function's `Cache-Control` / `Surrogate-Key` response headers) and the function's **fetch cache** (inner, controlled by backend headers or `new CacheOverride({ ttl })` / `{ mode: "pass" }`). Purging only the CDN makes the function serve stale data from its fetch cache, which the CDN then re-caches. Use the same surrogate keys in both and purge with the CDN Purge API **and** `aio aem edge-functions purge-cache <name> -k <key>` (or `purgeSurrogateKey()` in code). Publish-time invalidation from AEM does not reach the fetch cache.
- **Two deployments per change.** Adding an endpoint means a code deploy *and* a config-pipeline run for the new `cdn.yaml` rule; forgetting the second yields a 404 from publish, not from the function.
- **Smart deploy skips identical packages** — use `--force` when re-deploying the same hash on purpose.
- Backend `fetch` is open to any host by default; declare `origins:` in `edgeFunctions.yaml` and use `fetch(req, { backend: "name" })` to restrict.
- Runtime is Fastly Compute JS, not Node: no `fs`, no Node built-ins; check the [Fastly JS docs](https://js-compute-reference-docs.edgecompute.app/) before pulling npm packages.
