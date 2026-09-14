# ACS Commons Named Transform Image Servlet — runtime-readable transform ladder & model.json srcset

Context discovered while planning ticket 5252544 (Mayo AEM 6.5 LTS, acs-aem-commons 6.15.0, core components 2.30.4), Aug 2026.

## The servlet in one paragraph

ACS Commons' `NamedTransformImageServlet` serves resized derivatives of any DAM asset via a **suffix** URL — the transform name is a config key, not a parameter:

```
/content/dam/path/photo.png.transform/<name>/image.jpg
                              ^ asset path keeps its real extension; suffix appended after it
```

Each `<name>` is an OSGi **factory config** on `com.adobe.acs.commons.images.impl.NamedImageTransformerImpl`:

```xml
<jcr:root jcr:primaryType="sling:OsgiConfig"
          name="medium"
          transforms="[resize:width=904]"/>
```

A bogus name returns **404** — useful as a probe that a working URL is really the servlet and not a CDN wildcard. Output format follows the suffix filename (`image.jpg` → jpeg), regardless of source format — a PNG source gets converted (and compressed) for free.

## Key insight: the transform ladder is readable at runtime

`com.adobe.acs.commons.images.NamedImageTransformer` (the service interface each factory config registers) exposes the parsed config:

```java
public interface NamedImageTransformer {
    String getTransformName();                       // "medium"
    Map<String, ValueMap> getImageTransforms();      // {resize: {width: 904}}
    Layer transform(Layer);
}
```

So a Sling model / OSGi service can `@Reference(cardinality = MULTIPLE)` bind **all** `NamedImageTransformer` services and derive `{name → width}` — e.g. to build a `srcset` of transform URLs in model.json — with **zero duplication** between Java and the `ui.config` XMLs. The configs stay the single source of truth for widths; adding a rung to the ladder is config-only.

Caveat: **allowlist the names you emit.** Any feature can register more factory configs (a tiny `lqip`, social/OG-image sizes, …) and they would otherwise silently leak into every `srcset`.

Verification level: interface confirmed by `javap` on acs-aem-commons 6.15.0; the multi-bind pattern is **runtime-confirmed** (2026-08-14, Mayo ticket 5252544, AEM 6.5 LTS local author): a DS component with `@Reference(cardinality = MULTIPLE, policyOption = GREEDY)` on a `List<NamedImageTransformer>` field picked up a newly deployed factory config (`xlarge`) with zero Java changes — srcset gained the new rung on config deploy alone. Width values arrive as Strings in the ValueMap (parse, don't cast). Bonus finding: the servlet **upscales** sources smaller than the rung's width (1800px source → larger file at `resize:width=1920` than at 1440), so rungs above your typical original size cost bytes without quality.

## Free model.json plumbing via core components

Core components `Image` interface (2.30.4) already declares `default` methods `getSrcset()`, `getWidths()`, `getSrcUriTemplate()`, `getWidth()`, `getHeight()`. A delegating custom image model (`@Self @Via(ResourceSuperType)` pattern) only has to **override** them — the Sling Model Exporter serializes them into model.json automatically, no new API surface, and SPA consumers get a standard contract.

## Gotchas

- **jpeg-only in practice**: transforms destroy transparency (PNG), animation (GIF), and vectors (SVG) — guard those extensions out and fall back to the original URL.
- **AEM 6.5 has no server-side WebP/AVIF**: the `AssetDelivery` SPI is AEMaaCS-only (see [web-optimized-image-delivery.md](web-optimized-image-delivery.md)); named transforms resize/convert to jpeg/png/gif only. WebP needs the CDN (e.g. Akamai Image Manager).
- **CDN cache entries multiply**: each named variant is its own cached URL — asset re-publish purge must also purge `<assetUrl>.transform/<name>/image.jpg` for every name (on the Mayo stack: `AkamaiContentBuilder` appends the agent-configured `imageRenditionSuffix` values).
- **Dispatcher**: the request extension is the *asset's* extension but filters may also key on the suffix path; on the Mayo AMS setup the publish farm allowlists `transform` explicitly as an extension, and the assets farm admits it implicitly via a `/content/dam/*` allow. Transform *names* are not filtered — new named transforms need only an OSGi config, no dispatcher change.
- **vs Adaptive Image Servlet (`.coreimg`)**: `.coreimg` needs an image-policy width ladder mapped on every template, Java model support, and its page-path URLs are commonly blocked at dispatcher/CDN (403 on Mayo prod). Named transforms work on the DAM path directly — no template policy, no page path — which is why they can be "already working on prod" when `.coreimg` isn't.

Prod evidence (mayoclinic.org via Akamai, probed 2026-08-05): 893 KB PNG hero → 46 KB jpeg via `/small` (−95%); bogus transform name → 404.