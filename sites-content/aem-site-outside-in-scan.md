# Scanning a public AEM site from the outside (presales recipe)

How to learn an AEM site's platform, templates, libraries, martech and SEO state from the public site alone, before getting any access. Verified on a production on-prem/self-hosted AEM 6.x Sites site behind Akamai (presales scan, Oct 2026), using Chrome DevTools (MCP `evaluate_script`, `list_network_requests`).

## Use a real browser, not a fetcher

Sites behind **Akamai Bot Manager** (`bm_sv` / `ak_bmsc` cookies) return **403** to non-browser fetchers (WebFetch, plain curl without browser headers), while the same URL loads fine in Chrome. Run the checks with `fetch()` from inside a loaded page: same-origin, real cookies, no bot block.

Cross-origin `fetch()` (e.g. checking redirects on an old domain) fails on CORS. Do that one separately with curl.

## AEM fingerprints and what each tells you

| Signal | Where | Tells you |
|---|---|---|
| `/etc.clientlibs/<app>/clientlibs/...min.<hash>.js` | `<script src>` | AEM, the app name, and **whether clientlibs are versioned** (hash in URL = long cache TTL is safe) |
| `/etc.clientlibs/core/wcm/components/...` and `cmp-*` / `aem-Grid*` classes | Scripts, DOM | Core Components in use; list distinct `cmp-*` prefixes for a component inventory |
| `window.adobeDataLayer` entries | JS | **Core Components data layer leaks `repo:path` (real `/content/...` root) and `xdm:template` (the `/conf/.../settings/wcm/templates/...` path = editable templates)** plus component resource types |
| `<meta name="template">` | `<head>` | Template name per page; compare a content page vs a product page |
| 404 page text | Fetch any missing `*.xml` under a locale | The Sling default error page prints the resolved resource path (`Resource at '/content/<site>/us/en/x.html' not found`), so you see the content root and the URL mapping |
| `x-dispatcher` header | Response headers | Dispatcher host name, sometimes the region |
| `AWSALB` / `AWSALBCORS` cookies | Cookies | AWS Application Load Balancer, so self-hosted or managed on AWS |
| `akamai-grn`, `server-timing: cdn-cache`, `x-akamai-transformed`, `go-mpulse.net` / `akstat.io` | Headers, requests | Customer-run Akamai CDN, with mPulse RUM if Boomerang loads |
| `/libs/granite/csrf/token.json`, `/libs/cq/i18n/dict.<locale>.json` | Network | AEM Granite CSRF and the i18n dictionary are reachable on publish |
| `<company>.scene7.com/is/image/<company>/...` | Images | Dynamic Media (Scene7 mode); preset names like `:3-2-list` |
| Demdex subdomain, Scene7 company name, DAM folders of other brands | Network, DOM | **Origin of the platform**: after a spin-off or acquisition these still carry the parent company's name, which reveals an inherited multi-tenant codebase |

## One-shot inventory script (paste into DevTools or `evaluate_script`)

```js
() => {
  const g = ['jQuery','_satellite','s','Visitor','adobeDataLayer','dataLayer','google_tag_manager',
             'OneTrust','MktoForms2','bootstrap','moment','Handlebars','Vue','React','Granite','CQ'];
  const globals = {};
  for (const k of g) { try { const v = window[k]; if (v !== undefined)
    globals[k] = String((v && (v.fn?.jquery || v.version || v.VERSION || v.buildInfo?.buildDate)) || typeof v); } catch (e) {} }
  const hosts = {};
  for (const e of performance.getEntriesByType('resource')) { const h = new URL(e.name).host; hosts[h] = (hosts[h] || 0) + 1; }
  return {
    template: document.querySelector('meta[name=template]')?.content,
    canonical: document.querySelector('link[rel=canonical]')?.href,
    hreflang: [...document.querySelectorAll('link[hreflang]')].map(l => l.hreflang + ' ' + l.href),
    xDefault: !!document.querySelector('link[hreflang="x-default"]'),
    jsonld: document.querySelectorAll('script[type="application/ld+json"]').length,
    blockingScripts: [...document.scripts].filter(s => s.src && !s.async && !s.defer).length,
    stylesheets: document.querySelectorAll('link[rel=stylesheet]').length,
    cmp: [...new Set([...document.querySelectorAll('[class*="cmp-"]')]
      .flatMap(e => [...e.classList]).filter(c => c.startsWith('cmp-')).map(c => c.split('__')[0]))],
    dataLayer: (window.adobeDataLayer || []).slice(0, 2),
    globals, hosts: Object.entries(hosts).sort((a, b) => b[1] - a[1]),
  };
}
```

`globals._satellite` returns the **Adobe Launch library build date**. `s` returns the AppMeasurement version, and `Visitor` the ECID library version.

## SEO checks worth adding

- **Canonical target status:** `fetch(canonical)` and check `r.redirected`. A canonical that 301s to another URL (e.g. `/us/en/home` → `/us/en`) is a real finding that's easy to miss by eye.
- **Sitemap coverage:** parse `/sitemap.xml` `<loc>`s and count by `/<country>/<lang>/` prefix and by section (`/products/`). Check whether `xhtml:link` alternates exist. Compare the count with the audit's page counts.
- **hreflang vs locale folders:** the list in `<head>` is often shorter than the real locale set. Check a product (PIM) page separately: product templates often emit only their own language.
- **Localised pages linking to the master locale:** on a non-US home, count `<a href>` containing `/us/en/` and how many are absolute. A high share points to copied (not MSM-rolled-out) pages or hard-coded links.
- **Already-done items:** check `/llms.txt`, and search the raw HTML for removed scripts (a commented-out `<script>` still matches a naive grep).
- **Security headers** from one same-origin `fetch()`: look for CSP, Referrer-Policy, Permissions-Policy, and duplicate headers (e.g. `nosniff, nosniff` = set at two layers).

## Caveats

- **Consent state changes the tag set.** If the browser already accepted OneTrust (`OptanonAlertBoxClosed` cookie, `OptanonActiveGroups` lists all groups), you see every tag; a first-time or EU visitor may see fewer. Use an isolated browser context for a clean view.
- The **AEM version and service pack aren't exposed** on a hardened publish. Don't infer them from library versions.
- Not verified against an AEMaaCS site in this scan. There the CDN is Adobe's, so the Akamai/AWS signals above would differ; treat them as "6.5 / AMS / self-hosted" indicators, not proof.

Related: [[find-pages-by-template]], [[dispatcher-caching-and-invalidation]], [[language-root-properties]].
