# AEM Mock: `@Self @Via(ResourceSuperType)` delegation never reaches the Core Components model

## The gotcha

The Sling delegation pattern for extending a Core Component:

```java
@Model(adaptables = SlingHttpServletRequest.class, adapters = Image.class,
       resourceType = "myapp/components/image")            // supertype: core/wcm/components/image/v3/image
public class MyImageImpl implements Image {
    @Self
    @Via(type = ResourceSuperType.class)
    private Image image;                                    // expected: core v3 ImageImpl
    ...
}
```

works in AEM, but in a unit test using io.wcm AEM Mock **with the Core Components plugin** (`new AemContextBuilder().plugin(CORE_COMPONENTS).build()`) the injected delegate is **not** the core `ImageImpl` — it is a *second instance of your own model*, whose own delegate is `null` (Sling Models' recursion guard). Every delegated getter (`image.getFileReference()`, `getSrcset()`, …) therefore returns `null`, and tests that only assert nulls/fallbacks pass without ever exercising the delegation.

Registering the core model explicitly (`aemContext.addModelsForClasses(com.adobe.cq.wcm.core.components.internal.models.v3.ImageImpl.class)`) does **not** fix it. Neither does `aemContext.registerAdapter(SlingHttpServletRequest.class, Image.class, mock)` (any ranking) — Sling Models resolves `@Self` on a model interface through its own `internalCreateModel`, never via the AdapterManager.

## Why

The `CORE_COMPONENTS` plugin (`core.wcm.components.testing.aem-mock-plugin`) registers only two services: `DefaultPathProcessor` and **`LatestVersionImplementationPicker`**. That picker (`@ServiceRanking(1)`, "must come after `ResourceTypeBasedResourcePicker`") applies to any adapter type in `com.adobe.cq.wcm.core.components.models` and sorts candidates so that **classes outside `com.adobe.cq` come first**, then core internal models by descending version:

```java
.min((o1, o2) -> {
    Matcher m1 = INTERNAL_MODEL_PATTERN.matcher(o1.getName());
    Matcher m2 = INTERNAL_MODEL_PATTERN.matcher(o2.getName());
    if (m1.matches() && m2.matches()) return v(m2) - v(m1);          // latest core version
    return m1.matches() ? 1 : (m2.matches() ? -1 : 0);              // non-core class wins
})
```

In production, `ResourceTypeBasedResourcePicker` runs first and matches the supertype-forced resource type to the core `ImageImpl`, so the version picker is only a fallback. In the mock, the resource-type pick does not resolve for the wrapped request (observed on sling-mock 3.2.2 / aem-mock 4.1.8 / core components 2.30.4 — even a resource with `sling:resourceType=core/wcm/components/image/v3/image` and the core model registered adapts `Image.class` to the custom model), so `LatestVersionImplementationPicker` decides — and picks your class.

Verified empirically with a scratch test printing the injected field via reflection: delegate = `MyImageImpl@…`, `getSrc()` = `null`; direct `request.adaptTo(v3.ImageImpl.class)` **does** work (sling-mock auto-registers a `@Model` class on direct adaptation) and correctly resolves e.g. the page featured image.

## What to do

1. **Assert something the delegate must produce** at least once (e.g. `getSrc()` non-null for a resource with `fileReference`). If the existing test suite only asserts nulls, it is not testing delegation.
2. To unit-test logic that depends on the delegate, **inject a mock delegate** — the repo convention here is PowerMock's `Whitebox.setInternalState(model, "image", mock(Image.class))` after `adaptTo`. Adapter registration and model registration will not get you there.
3. Verify delegated behaviour that matters (page featured-image inheritance, policy-driven `sizes`, `srcset`) on a running instance via `model.json`.

## Related trap: the delegate resolves more than the component's own properties

Core Image v3 `initResource()` wraps the resource with **inheritance** (`Utils.getWrappedImageResourceWithInheritance`): with `imageFromPageImage` (the "Inherit featured image from page" checkbox; default on when the component has no `fileReference`/`file`) the effective `fileReference` comes from `<page>/jcr:content/cq:featuredimage`, and the component node has none. A custom model that reads `@ValueMapValue fileReference` for its own logic silently gets `null` in that mode — read `image.getFileReference()` from the delegate and fall back to the property.

Confirmed on the Mayo `MayoImageImpl` (Core Components 2.30.4, AEM 6.5 LTS), Aug 2026.