# Edge Delivery Services Configuration Service: site config via the Admin API

Where EDS site configuration lives now that `fstab.yaml` and friends are
retired, how a site is created and changed with `admin.hlx.page`, and which
changes are dangerous on production. Condensed from aem.live
[Setting up the configuration service](https://www.aem.live/docs/config-service-setup);
unauthenticated behaviour checked against a boilerplate site.

Companion notes: [[edge-delivery-project-anatomy]] (what stays in the repo),
[[edge-delivery-media-bus]] (content bus), [[edge-delivery-services-overview]].

---

## 1. Model

- **Site = `org` + `site`**, decoupled from GitHub `owner` + `repo`. Preview
  and live hosts use them: `https://<branch>--<site>--<org>.aem.page`.
  Several sites can point at one code repo with different content:
  **repoless** (aem.live `/docs/repoless`).
- **Inheritance**: organization → profiles → sites, so fleets of sites share
  settings and new sites need little config.
- **Consumers**: client (site + Sidekick), delivery, HTML pipeline, admin
  service. Only the **client scope** (public config, Sidekick) is readable
  from the outside; everything else is internal.
- **UI first**: [tools.aem.live](https://tools.aem.live) covers the common
  tasks; the REST API (`admin.hlx.page/config/…`, reference at aem.live
  `/docs/admin.html`) is for automation.
- **Legacy "document mode"**: projects from the `hlx.live` era still get a
  config aggregated from `fstab.yaml`, `.helix/config.xlsx` etc., for
  backward compatibility, but some features (repoless among them) need the
  Configuration Service.

## 2. Prerequisites

- **Every aem.live org must exist as a github.com org** with at least one
  repo synced by AEM Code Sync: that is how the namespace is claimed. Adobe
  or a partner may create the GitHub org, but **a customer employee must be
  an owner**.
- The tutorial flow (install Code Sync on a repo) **creates the aem.live org**
  with the GitHub org's name and makes the installing GitHub user its admin.
  Other admins: through Adobe.
- **Repoless needs one canonical site** whose `org/site` equals the GitHub
  `owner/repo`; code-to-config association and CDN push invalidation depend
  on it.

## 3. API recipes

All calls need `x-auth-token: <token>` (see aem.live
`/docs/admin.html#tag/authentication`); without it the config endpoints
return **401**. Base: `https://admin.hlx.page/config/<org>/sites/<site>`.

| Change | Method + path | Body |
|---|---|---|
| **Create site** | `PUT …/<site>.json` | `{"code":{"owner","repo"},"content":{"source":{"url":"https://content.da.live/<org>/<site>/","type":"markup"}}}`. Live at once on `main--<site>--<org>.aem.page`, serving `404.html` until content is previewed |
| Access control | `POST …/access/admin.json` | `{"role":{"config":["user@example.com"]}}` (role → users) |
| Code location | `POST …/code.json` | `{"owner","repo"}` |
| Content location | `POST …/content.json` | `{"source":{"url","type"}}` |
| Production CDN (push invalidation) | `POST …/cdn/prod.json` | `{"host","type":"fastly","serviceId","authToken"}` (other CDN types per aem.live `/docs/setup-byo-cdn-push-invalidation`) |
| Custom headers | `POST …/headers.json` | `{"/**":[{"key":"access-control-allow-origin","value":"*"}]}` (path glob → headers) |
| Public config | `POST …/public.json` | `{"custom":{…}}` → readable by anyone at `https://<host>/config.json` |
| Sidekick | `POST …/sidekick.json` | `{"plugins":[{"id","title","url"}]}` |
| Index | `POST …/content/query.yaml` | YAML, `content-type: text/yaml` |
| Sitemap | `POST …/content/sitemap.yaml` | YAML |
| robots.txt (prod domain) | `POST …/robots.txt` | plain text, `content-type: text/plain` |

Config can be written as a whole document (`<site>.json`) or per
sub-structure (the POST paths above), so one change does not require
resending the rest.

## 4. Traps

- **Never change `code` on a production site.** It disturbs the globally
  distributed caches. Create a new site with the new code, validate it, then
  move the **CDN origin** to the new site.
- **Changing `content` creates a new Content Bus ID**: every page must be
  **previewed and published again**, and the **index and sitemap configs**
  (stored with the content) are not carried over. Same fix: new site, set up
  content + index + sitemap, validate, switch the CDN origin.
- **`public.json` is public.** No secrets; it is served as `/config.json` on
  every host. With nothing configured, `/config.json` returns 404.
- **Config Service overrides repo/content files.** After switching, delete
  from GitHub: `fstab.yaml`, `robots.txt`, `tools/sidekick/config.json`,
  `helix-query.yaml`, `helix-sitemap.yaml`. When the `hlx.page` preview of the
  repo returns 404 (repo-based config cleaned up), unpreview and delete
  `.helix/config.xlsx` and `.helix/headers.xlsx` from the content.
- **Doc conflict on `robots.txt`**: the Anatomy page still allows a
  `robots.txt` in the repo root; this page lists it for removal once the
  Configuration Service is on. Prefer the config entry. Either way
  `*.aem.page`/`*.aem.live` always serve a disallow-all `robots.txt`.
- Reading a site's content source **does not need the config API**: the
  unauthenticated `admin.hlx.page/status/<org>/<site>/<branch>/<path>` returns
  `preview.sourceLocation` (see [[edge-delivery-project-anatomy]]).

## 5. What this means for AEM

The Configuration Service is EDS's equivalent of OSGi config plus
Dispatcher/CDN config in Cloud Manager, but there is **no pipeline**: a POST takes effect immediately on preview
and live. Treat the
Admin API like production infrastructure: script it, keep the request bodies
in version control, and use the "new site, then switch the CDN origin"
pattern for anything that touches code or content location.
