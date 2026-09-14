# `ReferenceProvider` aggregation: the undocumented `jcr:content` calling convention

`com.day.cq.wcm.api.reference.ReferenceProvider` is AEM's pluggable "what does this content
use" API: each product bundle registers its own OSGi service (XFs from
`cq-experience-fragments`, content fragments from `cq-dam-cfm-impl`, assets/media/policies
from `cq-wcm-core`, personalization, site manager, ...; ~8 on a stock author). No single
provider sees everything — a consumer collects ALL registered services and concatenates their
`findReferences(Resource)` results. This is what sits behind the publish wizard's
`/libs/wcm/core/content/reference.json?path=...`.

## The verified contract: pass `<path>/jcr:content`, fall back to the path

The interface javadoc says nothing about WHAT resource to pass. Decompiling the product
aggregator answers it — `com.day.cq.wcm.core.impl.reference.ActivationReferenceSearchBuilder
.addReferencesFromPath(...)` (cq-wcm-core 5.16.4) does, in bytecode:

```
getResource(path + "/jcr:content")   // constant pool: "/jcr:content" concat recipe
if null -> getResource(path)         // fallback for non-page/non-asset paths
for each provider: provider.findReferences(resource, onlyValidReferences)
```

So **Adobe's own caller always hands providers the `jcr:content` node when one exists,
never the page/asset node itself**. A custom aggregator must do the same
(`jcr:content` child if present, else the resource — a `jcrContentOrSelf` helper):

- Passing the **page node** is territory the product never exercises — provider behavior
  there is untested contract-free land. The concrete risk: a page node's subtree includes its
  **child pages**, so a property-scanning provider could report the whole subtree's
  references as the page's own (the XF provider does carry an internal `cq:Page`
  resource-type check in its walk, but nothing obliges other/custom providers to).
- Passing `jcr:content` scopes the scan to the page's own content by construction, works for
  `dam:Asset` items too (all CF data lives under the asset's `jcr:content`), and can't miss
  template-derived references — `cq:template` lives on `jcr:content`, and the XF provider
  resolves the editable template's structure XFs (header/footer) from there (verified
  empirically: `reference.json` for a WKND article returns `site/header/master` +
  `site/footer/master` as `xfvariations`).

Note there is also a 2-arg `findReferences(Resource, boolean onlyValidReferences)` variant
the product aggregator calls; the public interface method is the 1-arg form.

## What providers return (consumer must type-classify)

`Reference.getType()` is provider-chosen vocabulary, NOT a content-type system:
CF references arrive typed `"asset"` (indistinguishable from images without checking the
target's `jcr:content/contentFragment` marker or `adaptTo(ContentFragment.class)`), XFs
arrive as both `xfvariations` (the variation pages) and `experiencefragments` (the parent XF
pages), plus `template`, `contentfragmentmodel`, `caconfig` noise. Classify by inspecting the
returned resource, not by trusting `getType()`.

## Recipe: verifying product behavior by decompiling instance bundles

1. Map symbolic name → bundle id: `/system/console/bundles.json`.
2. Jar location: `<aem>/crx-quickstart/launchpad/felix/bundle<id>/version*/bundle.jar`
   (find `<aem>` via `lsof` on the quickstart java process).
3. `unzip` the jar (never grep it compressed), `javap -p -c` the class;
   `javap -v` shows the constant pool — string-concat recipes (`/jcr:content`) reveal
   path suffixes that `-c` output hides behind `invokedynamic makeConcatWithConstants`.

Same recipe family as [dialog-el-bindings.md](dialog-el-bindings.md).

Confirmed on AEM SDK 2026.6 local author (cq-wcm-core 5.16.4, cq-experience-fragments
1.3.116, cq-dam-cfm-impl 0.12.524), July 2026, for the Crowdin connector's reference
discovery service.