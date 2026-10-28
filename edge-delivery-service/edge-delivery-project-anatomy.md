# Edge Delivery Services project anatomy: repo, special files, configuration

What lives in an EDS code repo, which files are special, and what moved out
of the repo into the Configuration Service. Condensed from aem.live
[The Anatomy of a Project](https://www.aem.live/developer/anatomy-of-a-project);
claims about `aem.js` checked against `aem-boilerplate` 1.3.0.

Companion notes: [[edge-delivery-configuration-service]] (API recipes and
traps for the config below), [[edge-delivery-services-overview]],
[[edge-delivery-blocks-and-widgets]], [[edge-delivery-media-bus]] (code vs
content vs media bus).

---

## 1. Repo = website, per branch

- Buildless: the GitHub repo is served as-is once the **AEM Code Sync** app is
  installed. Every branch gets
  `https://<branch>--<repo>--<owner>.aem.page/` (preview content) and
  `…aem.live/` (published content).
- **Every committed file is public**: `/scripts/scripts.js` on `main` is
  `https://main--<repo>--<owner>.aem.page/scripts/scripts.js`. Adobe
  recommends public repos for the same reason: the code reaches browsers
  anyway.
- **Hostname limits**: `<branch>--<repo>--<owner>` must be **≤ 63 characters**
  (one DNS label), and none of the three parts may contain `--`. Long feature
  branch names silently hit this: keep branch names short.
- Non-GitHub Git hosts: "Bring your own git" (aem.live `/developer/byo-git`).

## 2. Special files in the repo

| File | Role | Rules |
|---|---|---|
| `head.html` | injected server-side into every page's `<head>`, merged with metadata from the content | keep near the boilerplate. Legitimate changes: **moving the code base to a subfolder** (repoint `scripts.js`/`styles.css`) or legacy non-module scripts. **No martech/GTM/Web SDK, no inline scripts or styles** (performance) |
| `404.html` | replaces the body of the default 404 for any URL matching neither content nor code | can reuse the site's header/footer code or look completely different. See aem.live `/docs/error-pages` |
| `.hlxignore` | `.gitignore` syntax; listed files are **not served and not observed** by the bot | use for tests, tooling, build artefacts, agent config, docs |
| `robots.txt` | served on the **production domain** only | `*.aem.page` and `*.aem.live` **always** serve a disallow-all `robots.txt`, whatever you configure. Repoless sites set it in the site config |

## 3. Configuration Service (not the repo)

Site configuration (content source, code location, indexing, sitemap,
robots, folder/CDN mappings, and optionally `head.html`) lives in the
**Configuration Service**, edited via the **AEM Admin API** or the UIs at
**tools.aem.live**.

- **Legacy files**: `fstab.yaml`, `helix-query.yaml`, `helix-sitemap.yaml`,
  `paths.json` in the repo, or config spreadsheets in the content, are from
  the file-based era. Adobe: in most cases they can just be deleted. Don't
  edit them expecting an effect.
- **Code connection**: written automatically when the Code Sync app is added;
  changed later via Admin API ("Update Code Location").
- **Content connection**: set at site creation, changed via Admin API
  ("Update Content Location"). To see it for a page, use the status endpoint
  (answered without a login for an unprotected site; protected sites need auth):
  `curl https://admin.hlx.page/status/<owner>/<repo>/<branch>/<path>` and read
  `preview.sourceLocation` (e.g. `markup:https://content.da.live/<owner>/<repo>/`
  = Document Authoring). The same response shows whether the page is
  previewed and published (`preview.status` / `live.status`).
- **Indexing**: an index definition (now in config) maintains a spreadsheet
  of pages for lists, feeds, filter/sort. **Sitemaps** are generated on
  publish, usually from the index, incl. hreflang.
- **Repoless**: several sites sharing one code repo; per-site `head.html` and
  `robots.txt` then live in the site config (aem.live `/docs/repoless`).

## 4. Conventional folders

| Path | Holds |
|---|---|
| `/scripts/aem.js` | the library (block loader, decoration, RUM). Vendored, don't edit |
| `/scripts/scripts.js` | project global JS; triggers decoration and block loading (eager → lazy → delayed) |
| `/scripts/consent-check.js` → `consented.js` | delayed phase. The boilerplate `consent-check.js` is a **dummy CMP that declines by default**, fires `consent.update`, and imports `consented.js` (martech, third-party code) only on consent. Real CMP wiring: aem.live `/developer/cmp-integration` |
| `/styles/styles.css` | global CSS needed for **LCP** only; keep small |
| `/styles/lazy-styles.css` | after LCP: fonts, below-the-fold global CSS |
| `/blocks/<name>/<name>.{js,css}` | most project code; name = folder = file = CSS class; JS is ESM with a default export |
| `/widgets/<name>/<name>.{html,css,js,json}` | application features dropped in by link; see [[edge-delivery-blocks-and-widgets]] |
| `/icons/<name>.svg` | authors write `:<name>:` |

`aem.js`, `scripts.js` and `styles.css` all load before first paint: every
byte there costs LCP.

**Icons, doc vs code**: aem.live says icons are "inlined into the DOM so
they can be styled with CSS". In `aem-boilerplate` 1.3.0 `decorateIcon`
inserts an **`<img src="/icons/<name>.svg">`** (16×16, lazy), which CSS
`fill`/`currentColor` cannot reach. Inline the SVG yourself (fetch + insert)
if icons must follow text colour.

## 5. Serving only part of a domain

When the CDN routes only, say, `/en/blog/` to EDS, the convention is to move
`/scripts`, `/styles`, `/blocks`, `/widgets`… into **`/en/blog/` in the
repo** and mirror `/en/blog/` in the content source, so no URL rewriting is
needed. Only `head.html`'s references change. `aem.js` derives
`window.hlx.codeBasePath` from the URL `scripts.js` was loaded from (the part
before `/scripts/`), so block, icon and font paths follow automatically, as
long as project code builds URLs from `codeBasePath` and not from `/`. When
EDS later takes over the whole domain, code moves back to the root.

## 6. What this means for AEM

An EDS repo replaces the Maven project, the `ui.apps`/`ui.config` packages,
Cloud Manager pipelines and Dispatcher config at once: the repo is the
deployment, and environment-level settings that used to be OSGi or
Dispatcher config are Configuration Service entries. None of the classic
deployment rules (immutable `/apps`, pipelines, quality gates) apply.
