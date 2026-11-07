# CORS for AEM headless on AEMaaCS: the OSGi policy vs. CORS at the dispatcher

Two Adobe AEMaaCS headless pages describe **two different, mutually exclusive ways** to let a browser app on another origin call GraphQL. Pick one per path. The generic policy mechanics (factory PID, first-match-wins, default deny, debugging) are in [[cors-policy-and-dispatcher]]; this note is the AEMaaCS headless delta.

Docs-based: Experience League "Cross-Origin Resource Sharing (CORS)" and "Dispatcher caching" (AEMaaCS headless deployment). Not reproduced on a live environment; inferences are flagged.

## Model A: the Granite CORS policy in AEM (no dispatcher caching)

File: **`com.adobe.granite.cors.impl.CORSPolicyImpl~<appname>-graphql.cfg.json`** in `ui.config`, deployed through Cloud Manager. It is a factory, so several policies can coexist and different ones can go to different environments (run-mode folders). Adobe's example (WKND), values as published:

```json
{
  "supportscredentials": false,
  "supportedmethods": [ "GET", "HEAD", "POST" ],
  "exposedheaders": [ "" ],
  "alloworigin": [ "https://my.domain" ],
  "maxage:Integer": 1800,
  "alloworiginregexp": [ "" ],
  "supportedheaders": [
    "Origin", "Accept", "X-Requested-With", "Content-Type",
    "Access-Control-Request-Method", "Access-Control-Request-Headers"
  ],
  "allowedpaths": [
    "/content/cq:graphql/wknd/endpoint.json",
    "/graphql/execute.json/.*"
  ]
}
```

- `alloworigin` or `alloworiginregexp` is mandatory; the trusted origin is the whole point.
- `allowedpaths` covers both the GraphQL endpoint (POST) and persisted queries (`/graphql/execute.json/.*`, GET). If the endpoint has a **vanity path**, add it too.
- **POST in `supportedmethods` is not enough for the endpoint.** CORS only lets the browser send the request and read the response; AEM's Sling Referrer Filter separately rejects the POST unless the origin's host is in `allow.hosts`. See [[sling-referrer-filter]]. Persisted queries over GET need only CORS.

Gotchas in the published example:

- **Endpoint path spelling differs between Adobe pages.** This page writes `/content/cq:graphql/...`; the AEM 6.5 CORS page writes `/content/_cq_graphql/...` (the URL-mangled form of the same JCR name). Which form the policy's path regex is matched against is *unverified*; if the endpoint is denied, try the other form or cover both.
- `allowedpaths` entries are regexes, and `endpoint.json` has an unescaped `.`. Harmless here, but escape (`endpoint\.json`) out of habit.
- `exposedheaders` and `alloworiginregexp` are `[""]`. Whether an empty regex in `alloworiginregexp` is skipped or matches every origin is *unverified*; use `[]` in your own files.

## Model B: CORS handled by the dispatcher (persisted-query caching on)

Full walkthrough of the flag, the vhost directives and the regex trap: [[graphql-persisted-query-dispatcher-caching]].

When persisted-query caching is enabled (`Define CACHE_GRAPHQL_PERSISTED_QUERIES` in the dispatcher `global.vars`; a rewrite adds a `.json` suffix so the responses become cacheable), Adobe says **the CORS filter is not needed and that section can be ignored**. CORS moves into the vhost:

- **The `Origin` header must NOT be forwarded to publish.** Check `clientheaders.any`. This is the opposite of Model A, which needs `Origin` to reach AEM so the policy can match. Mixing the two models is the classic broken setup.
- The vhost classifies requests with `SetEnvIfExpr` (`CORSType`, `CORSProcessing`, `CORSTrusted`), captures the origin with `SetEnvIfNoCase ^Origin$ ^https://(.*)$ CORSTrustedOrigin=https://$1`, and checks it against a regex placeholder `m#(https://.*.your-domain.tld(:\d+)?$)#` that you must adapt (escape its dots too).
- For a trusted origin the response gets `Access-Control-Allow-Origin` = the request's origin, `Access-Control-Allow-Credentials: true`, `Access-Control-Allow-Methods: GET`, `Access-Control-Max-Age: 1800`.
- An untrusted origin that needs CORS processing gets `RewriteRule "^(.*)" - [R=204,L]`.
- `Header merge Vary Origin` is set, but the dispatcher does not cache `Vary`.
- `FileETag Digest` in `<Directory />` gives each cached persisted-query response a unique ETag.

*Inference:* this is Adobe's answer to the tension noted in [[cors-policy-and-dispatcher]]. A cached file can carry only one `Access-Control-Allow-Origin`, so the header is computed per request at the dispatcher instead of being cached with the content, which makes multi-origin caching possible. Model B allows **GET only**, which fits persisted queries; an app still POSTing to the endpoint needs Model A for that path.

## Choosing

| Situation | Use |
|---|---|
| Persisted queries (GET), dispatcher caching on | Model B: vhost CORS, `Origin` not forwarded |
| GraphQL endpoint via POST (ad-hoc queries) | Model A policy **+** Referrer Filter entry |
| Both in one app | Model B for `/graphql/execute.json/*`; Model A only for the endpoint path |

## What this means for AEM

- **AEMaaCS:** both models apply. Configuration ships only through Cloud Manager (`ui.config` for the policy, the dispatcher module for the vhost). Adobe steers headless toward persisted queries over GET, which is also the path that avoids the Referrer Filter.
- **6.5 / AMS:** the same Granite policy exists and the 6.5 recipe in [[cors-policy-and-dispatcher]] applies. The `CACHE_GRAPHQL_PERSISTED_QUERIES` flag belongs to the AEMaaCS dispatcher configuration; on AMS the vhost technique can be reproduced by hand.

Related: [[cors-policy-and-dispatcher]], [[graphql-persisted-query-dispatcher-caching]], [[sling-referrer-filter]].

Sources: Adobe Experience League, AEMaaCS headless deployment pages "Cross-Origin Resource Sharing (CORS)" and "Dispatcher caching"; WKND reference project `ui.config/.../osgiconfig` on GitHub (adobe/aem-guides-wknd).
