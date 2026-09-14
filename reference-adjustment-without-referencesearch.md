# Reference discovery + adjustment without `com.day.cq.wcm.commons.ReferenceSearch`

The migration recipe for the deprecation described in
[aem-sdk-java-ceiling-dual-65-cloud.md](aem-sdk-java-ceiling-dual-65-cloud.md)
(`com.day.cq.wcm.commons` deprecated 2026-07-01, removal 2027-03-31, Cloud only).
Applies to any connector that must keep one branch working on **6.5 on-prem and Cloud**.

## Verified availability matrix

Checked by `unzip -l` on the actual artifacts (Sept 2026), not from docs:

| class | uber-jar 6.5.23 | aem-sdk-api 2026.8.27830 |
|---|---|---|
| `com.day.cq.wcm.commons.ReferenceSearch` | ✅ | ✅ (deprecated) |
| `com.day.cq.wcm.commons.utils.ReferenceSearch` | ❌ | ✅ |
| `com.day.cq.wcm.api.reference.ReferenceProvider` | ✅ | ✅ |

Two conclusions this settles empirically:

1. The package swap Cloud Manager's scan message prescribes (`…commons` → `…commons.utils`)
   is **not viable for dual support** — the class does not exist on 6.5 at all, so the bundle
   would fail OSGi resolution on-prem. This confirms the "Not an option" claim in the ceiling doc.
2. **`ReferenceProvider` is the safe API in both worlds** and is not deprecated — it is the
   correct foundation for anything reference-related in a dual-support connector.

`javap` on both classes shows the `adjustReferences` overloads are **identical** in the old and new
packages (the new one only *adds* `ResourceResolver`-carrying overloads and `setActionType`), so
there is no behavioral migration to reason about — purely a packaging/availability problem.

## The pattern that works: split discovery from rewriting

`ReferenceSearch` conflates two unrelated jobs. Once separated, neither needs the deprecated class.
Proven in the Crowdin AEM connector on a branch that explicitly supports 6.5 SP22 / 6.5 LTS.

### Half 1 — discovery → `ReferenceProvider` (product API, no reimplementation)

Bind every registered provider and aggregate, rather than using `ReferenceSearch.search()`:

```java
@Reference(cardinality = ReferenceCardinality.MULTIPLE,
           policy = ReferencePolicy.DYNAMIC,
           policyOption = ReferencePolicyOption.GREEDY)
private volatile List<ReferenceProvider> referenceProviders;
```

This is the same provider set behind the publish wizard. BFS the graph so references-of-references
are reached (page → template → its XFs), with a visited set to terminate cycles.
**Pass `<path>/jcr:content`, not the item node** — the calling convention and the
type-classification trap (`Reference.getType()` is provider-chosen vocabulary) are documented in
[wcm-reference-provider-contract.md](wcm-reference-provider-contract.md).

### Half 2 — rewriting → own code (the part worth reimplementing)

`adjustReferences(Node, from, to)` is a subtree walk rewriting `String`/`String[]` properties.
Reimplementing it is ~40–60 lines; the traps are what matter:

- **Match path boundaries, not raw prefixes.** `/content/site/en` must not corrupt
  `/content/site/en-gb`. This is the single highest-value test case — write it first, with exactly
  that pair.
- **Preserve property cardinality.** Read the value as `Object`, branch on `String[]` vs `String`,
  write back the same shape. A `ModifiableValueMap` will happily change a multi-value property into
  a single-value one.
- **Paths hide inside longer strings.** Rich-text `href`/`src` attributes need their own pass:
  regex the attribute, rewrite only the path portion, and byte-preserve everything around it.
  Preserve the `.html` extension and any `#fragment` / `?query` suffix.
- **Stop the walk at `cq:Page` children.** A child page is its own content item; its rich text is
  not this item's. Without this the walk rewrites content that belongs to another translation unit.
- **The stock implementation has a `/content/usergenerated` special case** (visible in its constant
  pool) — almost certainly irrelevant to a translation connector, but it explains behavior
  differences if you diff against the original.

### Two semantics to choose between

The naive reimplementation copies what `adjustReferences` does: **blanket prefix replace**
`sourceRoot` → `targetRoot`. It is the cheap like-for-like swap, but it happily rewrites links to
pages that were never translated, producing links to content that does not exist.

The more correct version resolves **each reference's actual target path** (stored translation
records first, then a derived-path service), and rewrites **only onto content that exists** or is
an active request item — leaving everything else at the source path. Cost: it needs a target-path
resolution service. Pair it with a post-pass that re-runs discovery and `WARN`s about any reference
still pointing at source-language content — cheap safety net, catches everything the rewrite missed.

Treat these as two tickets: the deprecation removal is urgent and mechanical, the semantics fix is
a correctness change that should not ride along with it.

Verified against the translated.com connector (single `adjustReferences` call site) and the Crowdin
connector's `ReferenceDiscoveryServiceImpl` / `ReferenceUpdateServiceImpl` /
`RichTextLinkUpdateServiceImpl`, Sept 2026.