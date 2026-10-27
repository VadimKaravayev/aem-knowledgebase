# Edge Delivery Services Block Collection and Boilerplate blocks

Which ready-made blocks exist for EDS, where their code lives, and how to
use them (copy the content model, then own the code). Condensed from aem.live
[Block Collection](https://www.aem.live/developer/block-collection); block
lists checked against the `adobe/aem-boilerplate` and
`adobe/aem-block-collection` repos on GitHub.

Companion notes: [[edge-delivery-blocks-and-widgets]] (how blocks work),
[[edge-delivery-project-anatomy]], [[adobe-skills-agent-plugins]]
(`block-collection-and-party` and `block-inventory` skills search these).

---

## 1. Three tiers

| Tier | Admission rule | Code | Support |
|---|---|---|---|
| **Boilerplate** | used by the **vast majority** of projects | [adobe/aem-boilerplate `/blocks`](https://github.com/adobe/aem-boilerplate/tree/main/blocks): every new project starts with it | Adobe, part of the product |
| **Block Collection** | used by **more than half** of projects, general enough to reuse **without changing the content model** | [adobe/aem-block-collection `/blocks`](https://github.com/adobe/aem-block-collection/tree/main/blocks) | Adobe, part of the product |
| **Block Party** | community showcase | links to community repos ([aem.live/developer/block-party](https://www.aem.live/developer/block-party/)) | **none**: Adobe does not maintain it, use at your own discretion |

Blocks move between tiers with usage: collection → boilerplate when
near-universal, boilerplate → collection (or deprecated and removed) when
usage drops. All are **content blocks**; application features (forms,
search results, calculators) are widgets and are not collected.

## 2. What is in each

**Boilerplate** (repo `blocks/`): `cards`, `columns`, `footer`, `fragment`,
`header`, `hero`, `widget`. The aem.live page lists Fragment under the
collection, but the boilerplate repo ships it too, plus the `widget` block
that backs `/widgets/` links. Projects forked earlier may lack `widget`.

Boilerplate also defines **default content** (no block needed): headings,
text, images, lists, links, buttons (bold/italic links), code, sections,
icons (`:name:`), plus the special tables **Metadata** (page metadata) and
**Section Metadata** (section styles/attributes).

**Block Collection** (in addition): Accordion, Carousel, Embed (social
media), Modal (**autoblock**: opened from a link, not authored as a table),
Quote, Search, Table, Tabs, Video, Breadcrumbs (**add-on** to the header,
no own block folder), **Form (deprecated)**. The collection repo also
carries copies of the boilerplate blocks, so it is a superset you can diff
against. Candidate under discussion: a consent banner.

## 3. How to use them

- **The content model is the value, the code is a starting point.** Adobe
  expects you to rewrite most of the CSS and JS. Blocks are **not versioned
  for compatibility and not upgradable**: copy them into `/blocks/<name>/`
  and own them; never treat the collection as a dependency.
- **Get the content structure** from the Sidekick Library on each block's
  aem.live page (copy button → paste into a document), or from the block's
  `.plain.html`. Keeping the collection's table shape means authors who know
  other EDS sites already know yours.
- **Check before building**: if a collection block's content model fits,
  start from it rather than designing a new table. If only the look differs,
  use block options (`Tabs (vertical)`) instead of a new block.
- **Quality bar** the collection follows, useful as a review checklist for
  your own blocks: intuitive to author; **no dependencies**, works with
  boilerplate; responsive at all breakpoints; **inherits CSS context** (text
  and background colour from the section); **no hard-coded text**
  (localizable); no performance cost; SEO-friendly and accessible.

## 4. What this means for AEM

The Block Collection plays the role Core Components play on the Java stack,
with one big difference: Core Components are a versioned dependency you
proxy and upgrade, while collection blocks are copied once and then belong
to the project. There is no "update to the latest block collection"; fixes
upstream have to be merged by hand if wanted.
