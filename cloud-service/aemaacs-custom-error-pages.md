# Custom Error Pages on AEMaaCS: the three layers

A 404 or 500 seen by a visitor on AEM as a Cloud Service can be produced by any of **three independent layers**, each with its own customisation mechanism. Customising only the AEM one is the usual mistake: the branded page then never shows when AEM itself is the thing that failed.

| Layer | Produces the page when… | Customised by |
|---|---|---|
| **Sling error handler** inside AEM | AEM answers the request with an error status via `sendError` | scripts under `/apps/sling/servlet/errorhandler/` |
| **Apache/Dispatcher** | AEM is unreachable or returns an error the Dispatcher is configured to replace | `ErrorDocument` directives in the Dispatcher vhost |
| **Adobe-managed CDN** | the CDN cannot reach the origin, or the origin returns a 5XX | `kind: CDN` config-pipeline YAML (`errorPages.spa`) |

---

## Layer 1: Sling error handler (`/apps/sling/servlet/errorhandler/`)

Default scripts live in `/libs/sling/servlet/errorhandler/` (`default.jsp`, `404.jsp`). Overlay them into `/apps/sling/servlet/errorhandler/` with the standard overlay mechanism, creating the folder if missing, and edit or add scripts there.

Script selection, from the Sling engine:

- **By status code**: `404.jsp`, `403.jsp`, `500.jsp` etc. Picked when the request ends with `HttpServletResponse.sendError(<code>)`.
- **By exception class**: a script named after the exception class or any superclass, resolved when an uncaught exception escapes a servlet or script. `Throwable.jsp` or `Exception.jsp` catches everything.
- `default.jsp` is the fallback.

Two documented traps:

1. **`500.jsp` is almost never run.** Adobe: it *"is only used if `HttpServletResponse.sendError(500)` is executed explicitly; that is, from an exception catcher. Otherwise, the response code is set to 500, but the `500.jsp` script is not executed."* An uncaught exception goes through the *exception-class* path, so a real 500 handler is `Throwable.jsp`, not `500.jsp`.
2. **`404.jsp` carries authentication logic.** Adobe: *"The 404.jsp script has been specifically designed to cater to AEM authentication; therefore, replacement of this script should be done with great care."* It is what turns a 404 for an anonymous user on a protected path into a login redirect. Copy it and change the rendering, do not write a fresh one.

### The WCM Debug Filter masks 500s on author

On **author** the *CQ WCM Debug Filter* (OSGi PID `com.day.cq.wcm.core.impl.WCMDebugFilter`) is enabled by default. It catches the exception and writes the full stack trace to the response **with status 200**, so the Sling error handler never fires and a custom 500 page cannot be tested there. Adobe: *"For a custom error-handler, responses with code 500 are needed - so the CQ WCM Debug Filter must be disabled."* On **publish** it is *"always disabled (even if configured as enabled)"*, so the custom handler works there untouched. Practical rule: disable the filter only on a dev author to test, via an OSGi config in the code base, and never expect a 500 page on author with it on.

### Sling-level facts the Adobe page assumes

- `response.setStatus(404)` does **not** trigger the error handler; only `sendError` does. A component that just sets the status renders the normal page body with an error code.
- The handler scripts run on publish as the request user, so a 404 page that includes site components must itself be readable by anonymous.
- A multi-site setup wanting per-site branded 404s typically does not hand-write the overlay at all but installs the **ACS AEM Commons Error Page Handler**, which resolves `/content/<site>/errors/404` from the requested path and dispatches to it from a thin `404.jsp`. The Adobe page does not mention it; it is the de-facto implementation.

## Layer 2: Dispatcher `ErrorDocument`

When AEM is down, or returns an error page you do not want to cache or expose, Apache serves its own document. In the Cloud Service dispatcher configuration this is an `ErrorDocument` in the vhost pointing at a static HTML file shipped with the dispatcher module. See [[url-resolution-layers-aemaacs]] for where the vhost sits in the request path. This layer never sees requests the CDN answers itself.

## Layer 3: CDN error page

Adobe: *"In AEM as Cloud Service, the CDN serves a generic error page when a 5XX error is received from the backend."* Two independent switches:

- **Let AEM's own error response through.** Add the response header `x-aem-error-pass: true` on the error response (typically from the Dispatcher vhost or the error script). Adobe: *"This works only for responses coming from AEM or the Apache/Dispatcher layer. Other unexpected errors coming from intermediate infrastructure layers will still display the generic error page."* Without the header, your carefully built Sling or Dispatcher 500 page is replaced by Adobe's generic one at the edge.
- **Brand the CDN's own page** for the case where the origin is unreachable. Config-pipeline file, e.g. `config/cdn.yaml`:

```yaml
kind: "CDN"
version: "1"
data:
  errorPages:
    spa:
      title: "Oops, something went wrong"
      icoUrl: "https://www.example.com/error.ico"
      cssUrl: "https://www.example.com/error.css"
      jsUrl: "https://www.example.com/error.js"
```

The CDN emits a shell page with that title, icon and stylesheet, an empty `<div id="root" status="<code>">` and your script tag; the SPA reads the `status` attribute and renders. The three assets must be hosted **outside AEM** (object storage, another CDN), because the page exists precisely for when AEM is unreachable. Deployed through a Cloud Manager **config pipeline**, not the code pipeline. Test without breaking anything via `https://publish-p<program>-e<env>.adobeaemcloud.com/cdnstatus?code=403` (supported codes 403, 404, 406, 500, 503).

## Decision list

- Visitor should see a branded 404 for a missing page → layer 1 (`404.jsp` overlay or ACS Commons Error Page Handler). Dispatcher and CDN are not involved for a 404 AEM answers itself.
- Visitor should see *your* 500 page when a component throws → `Throwable.jsp` in layer 1 **and** `x-aem-error-pass: true` on the response, or the CDN swallows it; test on publish or on an author with the debug filter off.
- Visitor should see a branded page when AEM is down → layer 3 SPA config; layer 2 covers the narrower case where Apache is up and AEM is not.
- "My custom 500 page works on publish but not on author" → not a bug, the WCM Debug Filter.
- "My custom 500 page works locally but the cloud shows Adobe's generic page" → missing `x-aem-error-pass` header.

## What this means for AEM

Layer 1 and the WCM Debug Filter behaviour are identical on 6.5; the same Adobe page exists for 6.5 minus the CDN note. Layers 2 and 3 are the Cloud Service additions: on 6.5/AMS there is no Adobe CDN switch, so the Dispatcher `ErrorDocument` is the outermost thing you control, and on Edge Delivery Services none of this applies (404 is the `404.html` in the project repo). Docs-based.

## References
- [Custom Error Pages, AEMaaCS (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/developing/full-stack/custom-error-page)
- [Configuring CDN Error Pages, AEMaaCS (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/content-delivery/cdn-error-pages)
- [CDN error pages tutorial (Adobe Experience League)](https://experienceleague.adobe.com/en/docs/experience-manager-learn/cloud-service/content-delivery/custom-error-pages)
- [Apache Sling error handling](https://sling.apache.org/documentation/the-sling-engine/errorhandling.html)
- [Overlays, AEMaaCS (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/developing/full-stack/overlays)
