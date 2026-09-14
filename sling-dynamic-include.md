# Sling Dynamic Include (SDI)

How to keep a page fully cacheable at the dispatcher/CDN while a few **dynamic** components (stock price, current time, FX rates, personalized fragments) stay fresh — without modifying the components themselves. Source: [Apache Sling — Dynamic Includes](https://sling.apache.org/documentation/bundles/dynamic-includes.html) and [Adobe — Set up Sling Dynamic Include for AEM](https://experienceleague.adobe.com/en/docs/experience-manager-learn/foundation/development/set-up-sling-dynamic-include).

---

## The problem it solves

A single uncacheable component poisons the cache for the **whole page**. If a footer stock widget pulls live data (or a slow third-party REST service), the dispatcher can't safely cache the page, so every visitor pays for the slow/dynamic render. The goal: cache the page once, fetch only the dynamic fragment separately. SDI pushes **cache granularity from page level down to component level**.

## What SDI is

An OSGi bundle (originally contributed by **Cognifide** to Apache Sling) that runs as a **servlet filter**. When a request's **resource type** matches the SDI config, the filter returns an **include placeholder tag** instead of rendering the component inline. The surrounding page is cached; the placeholder is resolved separately on each request.

**No component code changes are required** — SDI is wired purely via OSGi config + resource types.

## How it works (the `nocache` selector dance)

1. Request comes in for a page; SDI filter sees a component whose resource type is in its replace-list.
2. Instead of the real markup, SDI emits an **include tag** (SSI / ESI / JS) pointing at the same resource path but with a special selector injected — default selector name is **`nocache`**.
3. The page (now containing only placeholder tags for dynamic parts) is cached by the dispatcher/CDN.
4. The include layer (Apache, Varnish, or browser) issues a **second request** carrying the `nocache` selector.
5. SDI sees the marker selector, **passes the request through**, and the component renders its real, live content.

So: one cached page + N small dynamic sub-requests.

## Supported include types

| Type | Resolved by | Needs |
|------|-------------|-------|
| **SSI** (Server-Side Include) | Apache HTTP server / dispatcher | `mod_include` enabled |
| **ESI** (Edge-Side Include) | Varnish / CDN | ESI-capable edge |
| **JavaScript** | Browser (AJAX) | nothing extra — pure client-side |

The **JavaScript** variant is exactly the "load the component asynchronously via AJAX" pattern — placeholder renders immediately, real content fetched after page load. (JSON/binary output is supported too, but you must **disable the debug-comment option** for non-HTML.)

## Key configuration (OSGi factory config)

- Enable/disable toggle
- **Base path** the filter applies to (supports **regex** since v3.1.0)
- **Resource types** to replace
- **Include type** (SSI / ESI / JS)
- **Selector name** (default `nocache`)
- Component **TTL** (per-component cache lifetime at the edge)
- Required request headers
- URL parameter handling
- Debug-comment toggle (turn **off** for JSON/binary)

## Requirements

- AEM / Apache Sling 2+
- For **SSI**: Apache with `mod_include`. For **ESI**: Varnish or an ESI-capable CDN. For **JS**: nothing beyond the bundle.

## Limitations / gotchas

- **Incompatible** with components that handle **POST** requests, rely on **query strings**, or use **suffixes** — the include indirection breaks those.
- Components producing **non-HTML** output need the **debug-comment feature disabled** or the comment corrupts the payload.
- SSI/ESI require the dispatcher/CDN tier to actually process the include tags — if that layer doesn't, visitors see raw `<!--#include -->` / `<esi:include>` markup.

## When to reach for it vs alternatives

- **SDI (SSI/ESI):** best when the page should stay cached at the dispatcher/CDN and only a small fragment is dynamic. Server/edge-resolved, no client JS dependency.
- **SDI (JS) or hand-rolled AJAX lazy-load:** when you want the dynamic part fetched **client-side** after the page is interactive (good for slow/unreliable upstreams — keeps the slow call off the critical render path). This is the realistic implementation behind exam answers that say "load the component asynchronously."
- **Sling Jobs:** unrelated — those are for **background processing** (replication, integrations), not async *rendering*. Don't conflate.

> Note: "Sling Async Include" is **not** a real Apache Sling feature name. The documented mechanisms are **SDI** and the generic Servlet 3.0 async API. Use the correct terms.