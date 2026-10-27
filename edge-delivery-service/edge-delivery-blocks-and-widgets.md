# Edge Delivery Services blocks and widgets

How an authored table becomes a block, how `aem.js` decorates and loads it,
and when to build a widget instead. Condensed from aem.live
[Exploring blocks](https://www.aem.live/docs/exploring-blocks) and
[Widgets](https://www.aem.live/docs/widgets); loader behaviour read from
`scripts/aem.js` in `aem-boilerplate` 1.3.0 and checked against a running
boilerplate site.

Companion notes: [[edge-delivery-services-overview]],
[[edge-delivery-media-bus]] (where block images come from),
[[adobe-skills-agent-plugins]] (`building-blocks`, `content-modeling` skills).

---

## 1. Table in, divs out

A block is a **table** in the document (Word, Google Docs, or Document
Authoring). The first row's first cell is the block name; every further row
and cell is content.

```
| Accordion           |                 |      <div class="accordion">
| Heading one         |                 |  →     <div><div>Heading one</div></div>
| <image>             | Body text       |        <div><div><picture>…</picture></div>
                                                      <div>Body text</div></div>
                                                    </div>
```

- **Block name → class**: lower-cased, non-alphanumerics → `-`
  (`toClassName`). "Hero Banner" becomes `hero-banner`, which is also the
  folder name the loader looks for.
- **Options in brackets → extra classes**: `Columns (wide, dark)` gives
  `class="columns wide dark"`; commas separate options, while a multi-word
  option becomes one hyphenated class (`Columns (super wide)` →
  `columns super-wide`). The block name is always the **first** class;
  options are variants for CSS/JS, for differences too small to justify a
  new block.
- **One `<div>` per row, one `<div>` per cell.** A merged cell still produces
  a `<div>` per column, the extra ones **empty**. Authors add and drop rows and
  cells, so decorate defensively: never index cells by position without
  checking they exist.
- **Inspect the real markup** before writing code:
  `curl {host}/{path}.plain.html`. Section-level styles from Section Metadata
  appear there too (e.g. `<div class="highlight">` around a section).

## 2. What `aem.js` does with it

Order, inside `decorateMain` then section loading:

1. **`decorateBlock`** (only for `div.section > div > div`, i.e. top-level
   blocks; a table nested inside another block's cell is **not** decorated
   automatically):
   - adds class `block`, `data-block-name="<first class>"`,
     `data-block-status="initialized"`;
   - parent `<div>` gets `<name>-wrapper`, the section gets `<name>-container`
     (these are **section classes** — scope block CSS to `.<name>`, not to
     them);
   - **`wrapTextNodes`**: a cell whose first child isn't `p`, `pre`, `ul`,
     `ol`, `picture`, `table`, `blockquote` or `h1`–`h6` gets its content
     wrapped in a `<p>`. Bare text in a cell therefore arrives as `<p>`.
2. **`loadBlock`** when its section loads: fetches
   `/blocks/<name>/<name>.css` and dynamically imports
   `/blocks/<name>/<name>.js` **in parallel**, awaits `default(block)` (sync
   or async), then sets `data-block-status="loaded"`.
3. Sections load in order; the **first section is loaded eagerly** (LCP),
   the rest lazily, so a block in section 1 is on the performance-critical
   path.

Failure mode: a missing or throwing block JS is only a `console.error`
(`failed to load module for <name>`). The status still becomes `loaded` and
the raw divs render with whatever CSS loaded. A typo in the authored block
name looks like "unstyled content", not an error page.

`readBlockConfig(block)` turns a two-column key/value table into an object
(key via `toClassName`; value = link href(s), image src(s), paragraph
text(s) or plain text), for configuration-style blocks.

## 3. Decorating

The JS default export receives the block element and may: add classes/IDs,
swap in semantic elements, add ARIA attributes, restructure the DOM, attach
event handlers. Conventions that keep it robust:

- Work from the rows (`[...block.children]`) and test for each cell.
- Reuse authored elements (move `<picture>`s, don't rebuild `<img>` URLs);
  if you must build one, use `createOptimizedPicture` from `aem.js`.
- Keep all selectors under `.<name>`; only `fragment/fragment.js` is a
  sanctioned cross-block import, everything else shared goes in `/scripts/`.

## 4. Block or widget?

| | Block | Widget |
|---|---|---|
| For | adding form/function to **authored content** | **application** parts with little or no content: search results, dealer locator, calculator, sign-in, forms |
| Author does | fills in a table | pastes **one link** to the widget's `.html` |
| Files | `/blocks/<name>/<name>.{js,css}` | `/widgets/<name>/<name>.{html,css,js}` (+ optional `.json` config/data) |
| Markup from | the document | the widget's own `.html`, fetched and injected |
| Parameters | rows/cells, bracketed options | query string on the link → `data-*` attributes (`?name=World` → `data-name="World"`) |

Widgets work via an **auto block**: a paragraph containing only a link to
`/widgets/…/<name>.html` is replaced by a `widget` block, which fetches the
HTML, loads CSS and JS, and gives the element class `<name>` so it is styled
like a block. Keeping widgets out of `/blocks/` keeps the authors' block
library content-focused. Projects forked from an older boilerplate may not
have the `widget` block or auto-blocking yet: check `blocks/widget/` and
`buildAutoBlocks` in `scripts/scripts.js`.

## 5. What this means for AEM

Blocks are EDS's replacement for AEM components: no dialog, no Sling model,
no HTL. The content model is the table the author types, so changing a
block's structure is a content migration across every page that uses it.
That is why content modeling (rows/cells/options) is designed before code.
None of this applies to the AEMaaCS Java stack.
