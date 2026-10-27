# Edge Delivery Services code, content and media buses

Where each part of an EDS page is stored and served from, with the focus on
the **media bus** (images and video). Observed on a boilerplate-based EDS site
(`aem-boilerplate` 1.3.0, AEM CLI 16.x) plus aem.live docs.

Companion notes: [[edge-delivery-services-overview]] (what EDS is),
[[web-optimized-image-delivery]] (the Java-stack equivalent: `AssetDelivery`
and `AdaptiveImageServlet`).

---

## 1. Three buses, three sources

| Bus | Holds | Fed by | Changes go live by |
|---|---|---|---|
| **Code bus** | JS, CSS, `head.html`, icons, fonts: everything in the Git repo | GitHub, via the AEM Code Sync app on every push | merging to the branch (`main` = production code) |
| **Content bus** | page content (stored as markdown, rendered to HTML) | **preview** of a document in the content source (SharePoint, Google Drive, Universal Editor / AEM Author) | **publish**, separately from code |
| **Media bus** | binaries embedded in content (images, video) | the same preview step that ingests the document | publishing the page that references them |

The repo holds **no page content and no content images**. For "where is this
page in the repo?", the answer is that it isn't: only the code that decorates
it is. To see what the code receives, request `{path}.plain.html`
(`index.plain.html` for `/`).

## 2. Media bus mechanics

- **Ingested on preview.** Each image in the document is extracted, stored, and
  the markup rewritten to `./media_<hash>.<ext>`.
- **Content-addressed.** The name is a hash of the file's bytes. The same
  image on many pages is stored once. A replaced image gets a new name, so
  media URLs are effectively immutable and served with a long cache lifetime
  (observed `cache-control: max-age=2592000`, 30 days, versus `max-age=60`
  for the HTML).
- **Same origin as the page.** No separate asset host. Media resolves under
  any path of the site; the markup references it **relative** (`./media_…`).
- **Resized and converted on request** via query parameters:
  `?width=750|2000&format=webply|jpg|png&optimize=medium`. The `.plain.html`
  already contains a `<picture>` with WebP `<source>`s at 2000 px
  (`min-width: 600px`) and 750 px plus an original-format fallback. In code,
  `createOptimizedPicture(src, alt, eager, breakpoints)` in `scripts/aem.js`
  builds the same structure (default breakpoints: 2000 above 600 px, else 750).
- **LCP handling.** All authored images arrive `loading="lazy"`.
  `waitForFirstImage()` in `aem.js` switches the **first image of the first
  section** to `eager` and waits for it before loading the rest of the page.

## 3. Gotchas

- **Relative media paths break when content moves.** A fragment fetched from
  `/fragments/x.plain.html` and inlined into another page keeps `./media_…`
  URLs that would resolve against the host page. The boilerplate
  `blocks/fragment/fragment.js` rewrites `img[src^="./media_"]` and
  `source[srcset^="./media_"]` against the fragment path. Any custom block
  that fetches and inlines `.plain.html` needs the same rewrite.
- **Images committed to Git go to the code bus**, not the media bus. They are
  served as-is: no resize, no WebP, no content-hash URL, normal code-bus
  cache rules. Keep content images in the content source.
- **Local dev proxies media too.** `aem up` serves `localhost:3000/media_…` by
  proxying to the `.aem.page` origin; the response's `via:` header names the
  upstream `{branch}--{repo}--{owner}.aem.page`. Images are never local.
- **Changing an image is a content task.** No code change or merge alters it;
  replace it in the document and preview again.

## 4. What this means for AEM

The media bus is not AEM Assets: no DAM metadata, renditions, workflows or
rights management, and no `AssetDelivery`/`AdaptiveImageServlet`. Projects
that need governed assets reference them from AEM Assets (or Dynamic Media)
instead of embedding them in documents. For document-authored EDS sites,
the media bus is the whole image pipeline. It is irrelevant to the classic
AEMaaCS Publish + Dispatcher stack.

## References
- [aem.live: Markup, sections, blocks and auto blocking](https://www.aem.live/developer/markup-sections-blocks)
- [aem.live: Keeping it 100 (web performance)](https://www.aem.live/developer/keeping-it-100)
- [aem.live: Anatomy of a project](https://www.aem.live/developer/anatomy-of-a-project)
