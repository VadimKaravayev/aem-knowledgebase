# Edge Delivery Services development practices: GitHub flow, CSS, JavaScript, content first

Adobe's house rules for team development on EDS projects, condensed into a
checklist usable for PR reviews and agent rules files. Source: aem.live
[Development Collaboration and Good Practices](https://www.aem.live/docs/dev-collab-and-good-practices)
(docs-based; Adobe calls it lessons learned, not a spec).

Companion notes: [[edge-delivery-project-anatomy]],
[[edge-delivery-blocks-and-widgets]], [[edge-delivery-block-collection]],
[[adobe-skills-agent-plugins]].

---

## 1. GitHub flow (merge to `main` = production deploy)

- **Protect `main`**: PRs only, ≥ 1 approval, admins may bypass for
  emergencies. Code Sync deploys whatever lands on `main`.
- **Every PR links to the change running on its branch**:
  `https://<branch>--<repo>--<owner>.aem.page/<path>`, and for a changed
  block, a page that **uses that block** (the reviewer may not know where it
  appears).
- **Scope = the PR title.** Drafts for work in progress.
- **Talk to maintainers before opening a PR**, especially agent-written
  ones: for blocks that means agreeing the content model first. Adobe
  explicitly calls out "drive-by AI-slop PRs" with a large blast radius.
- **Lint clean before review** (`eslint` airbnb-base + `stylelint`, run in CI
  by the boilerplate). **Don't change lint config** for preference: it
  breaks copy-paste reuse from the boilerplate and Block Collection.
- **PageSpeed Insights** runs on each PR via Code Sync: **green for mobile
  and desktop, ideally 100**, before asking for review. Automatic PSI checks
  on feature branches are **not enabled for private repos**.
- **Branches are private to their author**; push to someone else's only when
  invited. **Authors merge their own PRs**, and check with the author before
  merging even an approved PR.
- **Scaled trunk-based development**: small PRs merged often; avoid
  long-lived branches.
- Keep the (few) devDependencies current; Renovate is fine.
- **Public repo recommended**: code ships to browsers anyway. Private repos
  lose free branch protection on GitHub Free and the automatic PSI checks.

## 2. CSS

- **Every selector in `blocks/<name>/<name>.css` applies only inside
  `.<name>`.** Blocks share one DOM and CSSOM.
- Prefix block-private classes and custom properties with the block name;
  **don't** prefix classes/variables meant to be inherited or authored
  (section styles, theme variables).
- No reformatting or property reordering in functional PRs; keep the diff to
  the change.
- Prefer adding a class in JS over a complex selector. Simple names, no
  design-system namespacing unless needed.
- `!important`: practically never.
- **Style on ARIA state** (`[aria-expanded="true"]`, `[aria-hidden]`) rather
  than inventing `.open`/`.is-active` classes.
- **Mobile first; breakpoints `600px`, `900px`, `1200px`, all `min-width`**;
  never mix `min-width` and `max-width`.
- No Sass/Less/PostCSS/Tailwind without team- and org-wide agreement; use
  native CSS that evergreen browsers support.

## 3. JavaScript

- No framework for layout problems (LCP/TBT cost); frameworks only for
  application-like features, and only with team buy-in. Same for build
  tooling: **stay buildless**.
- **Browser baseline = browsers with dynamic `import()`** (what `aem.js`
  needs); anything those support is safe. Note some syntax fails the whole
  module, not just one feature (optional chaining is the classic example).
- **Third-party libraries**: never in `head.html`; load with `loadScript()`
  inside the block that needs them, and for big ones only when the block
  scrolls into view (`IntersectionObserver`).
- **Don't modify `aem.js`** (`adobe/aem-lib`, shipped unminified for
  debugging); put project extensions elsewhere, send general fixes upstream.
- **`<head>` stays minimal**: no martech, no inline scripts or styles.
- **Don't minify.** Code is served per block, compressed, over HTTP/2; Adobe
  says minification gives no measurable gain unless files are huge (itself an
  anti-pattern), and costs a build step plus source maps.

## 4. Content first

- **Design the content before the code**: build sample content in the
  document/spreadsheet, have someone with author-support experience review
  it, then write CSS/JS.
- **Use `/drafts/<name>/`** for content that goes with unmerged code; authors
  move it into place after the merge. Content and code have separate
  lifecycles.
- **Backwards compatible by default**: merging code must not change existing
  pages or force content refactoring; new behaviour appears when new content
  is previewed and published (design changes and bug fixes excepted).
- **No binaries or content in Git.** Images, HTML pages and SVGs belong in
  content; an SVG in the repo is justified when code references it (icons).
  A PR full of images or HTML files is a red flag. (Widgets' own `.html`
  markup is code, not content, and is the sanctioned exception.)
- **No visitor-facing string literals in JS or CSS**: source them from
  content (placeholders spreadsheet, aem.live `/docs/placeholders`) so they
  can be edited and translated.

## 5. What this means for AEM

Most of this is ordinary frontend hygiene; the EDS-specific weight comes
from **merge = deploy** (no pipeline or stage between a PR and production)
and **content and code shipping independently**. That is why the review
gate, the branch preview link, the PSI check and backwards-compatible
content models carry the load that Cloud Manager's quality gates and stage
environment carry on the Java stack.
