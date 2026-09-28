# AEM Image Delivery: AdaptiveImageServlet & AssetDelivery SPI

AEM Cloud Service provides a platform-level `AssetDelivery` SPI that serves DAM assets through an optimized CDN with automatic WebP negotiation. Core Components' Image component exposes this via the "Enable Web Optimized Images" design dialog checkbox. Custom components can use the same SPI directly.

---

## How it works

1. **`com.adobe.cq.wcm.spi.AssetDelivery`** — single-method interface in `aem-sdk-api`. AEM Cloud Service provides the implementation at runtime. Local SDK may or may not have it.

2. The interface:
   ```java
   public interface AssetDelivery {
       String getDeliveryURL(Resource resource, Map<String, Object> parameterMap);
   }
   ```
   Takes a DAM asset `Resource` + parameter map, returns an optimized delivery URL.

3. The CDN handles format negotiation (WebP when browser supports it via `preferwebp=true`) and server-side transformations (crop, rotate, flip, resize, quality).

## URL pattern

Normal (AdaptiveImageServlet):
```
/content/page/jcr:content/image.coreimg.82.1280.jpeg/1234567/photo.jpeg
```

Web-optimized (AssetDelivery):
```
/asset/delivery/content/dam/photo.jpg.photo.jpg?width=1280&quality=82&preferwebp=true
```

## Parameters

| Key | Value | Purpose |
|-----|-------|---------|
| `path` | DAM path string | Asset location |
| `seoname` | string | SEO-friendly filename in URL |
| `format` | `jpeg`, `png`, `gif` | Original format (CDN may serve WebP instead) |
| `preferwebp` | `"true"` | Auto-serve WebP when browser supports it |
| `width` | pixel string | Desired width |
| `quality` | 0–100 string | JPEG quality |
| `c` | crop rect | Percentage (`x%,y%,w%,h%`) or pixel (`x,y,w,h`) |
| `r` | degrees string | Rotation |
| `flip` | `HORIZONTAL` / `VERTICAL` / `HORIZONTAL_AND_VERTICAL` | Flip |

## How Core Components uses it

In `ImageImpl` (v1):
```java
useAssetDelivery = currentStyle.get(PN_DESIGN_ASSET_DELIVERY_ENABLED, false) && assetDelivery != null;
```

- Reads the policy property `enableAssetDelivery` (boolean, default false)
- Only enables if the `AssetDelivery` OSGi service is present
- Delegates to `AssetDeliveryHelper` (internal class) which builds the param map and calls `getDeliveryURL()`
- Falls back to `AdaptiveImageServlet` URL pattern if disabled or service unavailable

The design dialog checkbox has a render condition (`core/wcm/components/rendercondition/isAssetDeliveryEnabled`) that hides it when the platform doesn't provide the service.

## Using it in custom components

`AssetDeliveryHelper` is in Core Components' `internal` package — not importable. But the `AssetDelivery` SPI is one method, so the pattern is trivial to replicate:

```java
@Model(adaptables = SlingHttpServletRequest.class)
public class MyComponentModel {

    @OSGiService(injectionStrategy = InjectionStrategy.OPTIONAL)
    private AssetDelivery assetDelivery;

    @ScriptVariable
    private Style currentStyle;

    private String imageUrl;

    @PostConstruct
    protected void init() {
        boolean useAssetDelivery = currentStyle.get("enableAssetDelivery", false)
                && assetDelivery != null;

        String fileReference = resource.getValueMap().get("fileReference", String.class);

        if (useAssetDelivery && StringUtils.isNotEmpty(fileReference)) {
            Resource assetResource = resource.getResourceResolver().getResource(fileReference);
            if (assetResource != null) {
                Map<String, Object> params = new HashMap<>();
                params.put("path", fileReference);
                params.put("seoname", "my-image");
                params.put("format", "jpeg");
                params.put("preferwebp", "true");
                params.put("width", "1280");
                params.put("quality", "82");

                imageUrl = assetDelivery.getDeliveryURL(assetResource, params);
            }
        }

        if (StringUtils.isEmpty(imageUrl)) {
            imageUrl = fileReference; // fallback
        }
    }
}
```

### Design dialog checkbox for custom components

Add to `_cq_design_dialog/.content.xml`:
```xml
<enableAssetDelivery
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/checkbox"
    checked="${cqDesign.enableAssetDelivery}"
    name="./enableAssetDelivery"
    text="Enable Web Optimized Images"
    uncheckedValue="false"
    value="{Boolean}true">
    <granite:rendercondition
        jcr:primaryType="nt:unstructured"
        sling:resourceType="core/wcm/components/rendercondition/isAssetDeliveryEnabled"/>
</enableAssetDelivery>
```

## Adobe documentation

https://experienceleague.adobe.com/en/docs/experience-manager-core-components/using/developing/web-optimized-image-delivery

## Gotchas

- **AEMaaCS only** — local SDK may not have the `AssetDelivery` service. Always check for null. The render condition hides the checkbox when absent.
- **SVGs excluded** — Core Components skips SVG extension. Asset Delivery is for raster images only.
- **DAM assets only** — inline-uploaded images (no `fileReference`) can't use this. Fall back to AdaptiveImageServlet.
- **Crop coordinates** — if author-applied crop exists, you need the web rendition's dimensions to convert pixel crop to percentage. See `AssetDeliveryHelper.getCropRect()` for the math.
- **No Dynamic Media overlap** — ImageImpl disables Asset Delivery when DM features are enabled. Pick one or the other.

---

## Adaptive Image Servlet (fallback path)

The `AdaptiveImageServlet` is the server-side image delivery path — it serves images when `AssetDelivery` is disabled or unavailable. It handles resize, crop, rotate, flip, and quality transformations on the AEM instance itself.

Source: `bundles/core/.../internal/servlets/AdaptiveImageServlet.java`
Adobe docs: https://experienceleague.adobe.com/en/docs/experience-manager-core-components/using/developing/adaptive-image-servlet

### URL pattern

```
/content/page/jcr:content/mycomponent.coreimg.{quality}.{width}.{extension}/{lastModified}/{seoname}.{extension}
```

Selector forms: `handler` | `handler.width` | `handler.quality.width`

The servlet validates that the requested width exists in the policy's `allowedRenditionWidths` — non-allowed widths are rejected (the "Invalid image request" 404).

### Registering for custom resource types

The servlet uses an OSGi configuration factory (`AdaptiveImageServletMappingConfigurationFactory`) that maps `{resource types, selectors, extensions}` → servlet. Out of the box:

- `core/wcm/components/image` + selector `img` (v1 URLs)
- `core/wcm/components/image` + `cq/Page` + selector `coreimg` (v2/v3 URLs)

You can create an additional factory config to register your custom component's resource type. The servlet then serves images for it with the same transformation logic.

### Policy properties the servlet reads

| Property | What it controls |
|----------|-----------------|
| `allowedRenditionWidths` | Which widths are valid in the URL (**required** — without this, URLs 404) |
| `jpegQuality` | Quality percentage (default: 82) |
| `enableAssetDelivery` | If true + service available, `ImageImpl` bypasses this servlet entirely |

### Rendition selection algorithm

1. Collects all DAM renditions of the asset
2. Filters by matching MIME type (PNG original → PNG renditions preferred)
3. Sorts by width ascending
4. Picks the smallest rendition ≥ requested width (avoids upscaling)
5. Falls back to original if no suitable rendition exists

### Key behaviors

- **Never upscales** — if requested width > original, serves at original size
- **GIF/SVG pass-through** — no transformation, streams original binary
- **TIFF → JPEG** — auto-converts TIFF to JPEG
- **Max input size** — rejects images wider than 3840px (configurable via `maxSize`) to prevent OOM
- **`If-Modified-Since` caching** — returns 304 when asset hasn't changed; requires dispatcher config to pass the header through
- **Crop/rotate/flip** — reads `imageCrop`, `imageRotate`, `flipHorizontal`/`flipVertical` from component properties (authored via Image editor)
- **Transparent PNG → JPEG** — adds white background when converting transparent images to JPEG

### Using the servlet for custom components

Three approaches, simplest first:

**1. Supertype from Core Image** — your component extends `core/wcm/components/image/v3/image`. You inherit both the servlet registration and `AssetDelivery` support for free. Best when your component is primarily an image.

**2. OSGi factory config** — create a config that registers your resource type with the servlet. You build the `coreimg`-style URL in your Sling Model and the servlet handles transformation. Needs a policy with `allowedRenditionWidths`.

**3. AssetDelivery SPI directly** — inject `AssetDelivery`, build the param map, get CDN URLs. No servlet involved. Best when your component has images alongside other fields and can't supertype Image. See the "Using it in custom components" section above.

### Defaults

| Setting | Default | Source |
|---------|---------|--------|
| JPEG quality | 82 | `AdaptiveImageServlet.DEFAULT_JPEG_QUALITY` |
| Default resize width | 1280 | `AdaptiveImageServlet.DEFAULT_RESIZE_WIDTH` |
| Max input width | 3840 | `AdaptiveImageServlet.DEFAULT_MAX_SIZE` |
