# Classifying a picked path: page, XF, content fragment or asset

A translation connector (or any tool that acts on a mixed set of picked paths) has to answer
"what kind of content is this?" for every selection, because the answer decides what gets
extracted, how it is serialized and where the result is written back. The obvious approach —
a table of JCR primary type plus path prefix, one row per type — is the one that breaks.

## The predicates are nested, so order decides, not the rules

Written as independent definitions they overlap in two places, and **both overlaps are total,
not edge cases**:

| Type | Naive rule | Overlaps with |
|---|---|---|
| Content fragment | `dam:Asset` under `/content/dam` **with** a CF marker | every CF is also an asset |
| Asset | `dam:Asset` under `/content/dam` | — |
| Experience fragment | `cq:Page` under `/content/experience-fragments` | every XF is also a `cq:Page` under `/content` |
| Page | `cq:Page` under `/content` | — |

So a rule set is not a partition and cannot be evaluated in an arbitrary order. What an
implementer needs is an **ordered ladder, first match wins**, most specific first:

1. content fragment
2. asset
3. experience fragment variation
4. site page

Get the order wrong and the failure is silent: a fragment classified as an asset exports its
metadata instead of its fragment elements, and an XF classified as a page is written back to
the wrong root.

## Use the product APIs, not path and property matching

Every step of the ladder has a product API that is both shorter and more robust than a JCR
query, and each carries **containing-item semantics**: a hit *inside* an item types as the
item, which is what you want when a picker or a search returns `jcr:content` or a deeper node.

```java
// 1 + 2: the containing dam:Asset, then the CFM marker on its jcr:content
Asset asset = DamUtil.resolveToAsset(resource);              // com.day.cq.dam.commons.util.DamUtil
Resource content = resolver.getResource(asset.getPath() + "/jcr:content");
boolean isCf = content != null && content.getValueMap().get("contentFragment", false);

// 3 + 4: the containing page, then the XF adapter
Page page = resolver.adaptTo(PageManager.class).getContainingPage(resource);
boolean isXf = page.adaptTo(ExperienceFragmentVariation.class) != null;  // com.adobe.cq.xf
```

- **`contentFragment` (boolean, on the asset's `jcr:content`) is the marker, not `cq:model`.**
  It is the persistent contract the CFM adapter itself checks. `cq:model` is a trap: it is not
  on the `dam:Asset` node and not on `jcr:content` — it sits at `jcr:content/data/cq:model`, so
  a rule or QueryBuilder predicate written against `jcr:content/cq:model` matches **nothing**
  and every fragment silently degrades to a plain asset. Verified on AEM 6.5 with WKND:
  `jcr:content/cq:model` → 0 results, `jcr:content/data/cq:model` → all 49 fragments.
- **`adaptTo(ExperienceFragmentVariation.class)` beats a path prefix.** Only variation pages
  adapt; XF *parent* pages and folders do not, which is exactly the distinction you want, since
  only a variation is a translatable unit. It also sidesteps the common spec typo
  `/content/experience/fragments` — the real root is `/content/experience-fragments`, one
  segment with a hyphen, and a wrong prefix means every XF falls through to Page.

## Restrict the page step to the site tree

`cq:Page` nodes also live outside `/content`, notably under `/conf` as template *initial
content*. And non-variation pages inside the XF tree are not translatable units. So the last
rung is not "is a page" but:

```java
page.getPath().startsWith("/content/")
    && !page.getPath().startsWith("/content/experience-fragments/")
```

Anything that falls off the end is `UNKNOWN` — return that rather than guessing, and let the
caller refuse the item with a message.

## Per-type pickers remove the ambiguity from the UI, not from the code

If each content type opens its own picker — AEM's stock CF picker at
`/libs/dam/cfm/content/cfpicker/picker` lists folders and fragments only, its column view via
`dam/cfm/components/cfpicker/datasources/children` and its search via hidden form fields
`type=dam:Asset` + `contentfragment=true` — then the author never sees a list where the
question could arise. That is the right UI answer (see
[granite-foundation-picker-control.md](granite-foundation-picker-control.md) for cloning it).

It does **not** remove the need for the ladder. The connector still resolves a type per item
after the pick, for serialization and write-back, and any path can also arrive from a saved job,
a REST call or a workflow rather than from the picker.

Confirmed against the Crowdin AEM connector's `ContentTypeServiceImpl`
(`resolveContentType`: CF → asset → XF variation → site page) and its CF picker clone at
`/apps/crowdin/content/cfpicker/picker`, plus a local AEM 6.5 author with WKND sample content,
September 2026.