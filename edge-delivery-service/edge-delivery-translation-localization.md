# Edge Delivery Services translation and localization: folder strategy, DA Translate app, placeholders

How to make an EDS site multilingual: the URL/folder strategy Adobe
recommends, translating documents with the Translate app in Document
Authoring (DA, da.live; newer aem.live docs call it "Experience Workspace"),
its `.da/translate` config, UI strings via placeholders, and the code a
boilerplate project still needs. Condensed from aem.live (pages listed under
References); repo facts checked against `aem-boilerplate` 1.3.0 and
`adobe/aem-block-collection`.

Companion notes: [[edge-delivery-configuration-service]] (sitemap/hreflang,
index), [[edge-delivery-dev-good-practices]] (no literals in code),
[[edge-delivery-block-collection]] (`placeholders.js` lives there),
[[edge-delivery-blocks-and-widgets]].

---

## 1. Folder strategy: language first, market second

Adobe recommends `/<lang>/` then `/<lang>/<market>/`:

```
/en/   /en/us/   /en/apac/   /es/us/   /es/mx/
```

- A market folder holds **only what really differs** for that market:
  homepage, campaign pages, legal pages, market settings, localized `nav`
  and `footer`. Everything else is served from the language folder.
- Market-first trees (`/ca_fr/`, `/en-us/` + `/en-ca/`) duplicate content,
  hurt SEO (search engines pick canonicals for you) and lock visitors out
  of languages their market doesn't list.
- A root fallback is fine (`/` = English for a US-centric business).
- Spreadsheets follow the same tree: a market `placeholders` sheet overrides
  the language one.

| Strategy | Example | Use when |
|---|---|---|
| **Language-based** (default recommendation) | `/en` `/fr` `/de` | content is the same everywhere, only the language differs |
| **Region-based** | `/ch/de` `/ch/fr` `/ch/it` | content is regional (typical for ecommerce) |
| **Locale-based** | `/en-us` `/fr-ca` | simple for commerce, but duplicates content |
| **Hybrid** | `/en`, `/en/us`, `/fr/ca` | mostly language-based with some regional pages; preferred over locale-based |

**Detection.** Language and market are separate. Visitors from search, paid
or social already arrive in a language; browser language is only a fallback
for new visitors with no other signal. Market from geo IP or the Geolocation
API, always overridable with a market switcher, persisted on the device.
Some legislation forbids geofencing, so never force it.

## 2. Translating documents: the Translate app (DA)

Works with **no setup**, using **Google Translate**. Open
`https://da.live/apps/loc`.

1. **Setup**: title + list of page URLs. Pages must be **previewed or
   published** and in **one site**. *Validate sources* checks them and
   **adds referenced fragments automatically** (fragments from other origins
   are not added).
2. **Confirm options**: environment (default `prod`), conflict behaviour,
   and an action per language:
   - **Translate**: send to the service, save into the language folder;
   - **Copy**: source content into the language folder, untranslated;
   - **Rollout**: copy from the language folder to its locale folders only;
   - **Skip**: always available.
   A language may be translated **from another language** (`fr-CA` from
   `fr-FR`); it waits until its source is saved.
3. **Sync** (only if some URLs are outside the source folder, e.g. `/en`):
   bring them into the source folder, or skip.
4. **Translate** → *Get status* → saved into `/de/`, `/fr/`…; then
   **Rollout** to locales if configured → *Complete project*.

**Conflicts** (per step: sync, translation return, copy, rollout):
`overwrite` or `merge`. Merge shows the differences inline, remembers
accepted/rejected decisions, and is how **regional edits survive repeated
rollouts**. **Spreadsheets/JSON are always overwritten**; a page with
nothing to merge against is overwritten; identical content is not written.

Every save is a **page version labelled with the project title**
(restorable). Projects are listed on the Localization Dashboard (filter,
archive, copy a project's parameters into a new one).

## 3. Configuration: the `.da/translate` sheet

Create a sheet `translate` in the site's hidden folder
(`https://da.live/#/<org>/<site>/.da`), rename its first tab to `config`,
use `key` / `value` columns. Unset keys keep their defaults. Reading it
via `admin.da.live/source/…` needs a DA login (401 otherwise).

**`config` tab** (selection):

| Key | Meaning |
|---|---|
| `translation.service.name` | `Google` (default), `Lionbridge`, `Smartling`, `Trados` |
| `translation.service.all.env` | environments offered, first = default (`stage, prod`) |
| `translation.service.{env}.*` | per-environment endpoint and credentials (`origin`, `apiEndpoint`, `authEndpoint`, `clientId`, `clientSecret`, …) |
| `translation.service.key.path` | Admin API URL of a **separate credentials sheet** `.da/translate-keys.json` (recommended: keep secrets out of the main sheet) |
| `sync./translate./copy./rollout.conflict.behavior` | comma-separated choices, **first = project default** (`merge, overwrite`) |
| `source.language` / `source.language.detect` | source-of-truth language row / derive it per project from the URLs' folder |
| `complete.aem.preview` / `complete.aem.publish` | auto preview/publish finished pages **unless merge conflicts remain** |
| `translation.service.custom.{type}.{name}` | extra project options rendered by a connector |

**`languages` tab**: `name`, `code`, `location` (folder), `actions`
(Translate, Copy, Rollout), `locales` (rollout targets), `default locales`,
`source` (path override: URLs under it don't trigger sync),
`source language` (depends on another row), plus Trados-specific columns.

**Do-not-translate tabs**:
- `custom-doc-rules`: per block, e.g. `section-metadata` → `do-not-translate`;
  `metadata` → `translate col 2 if col 1 is "title" or "description"`;
  `columns` → `dnt col -1`.
- `dnt-content-rules`: exact strings never translated (product names, paths,
  colour values).
- `dnt-sheet-rules`: patterns for spreadsheet/JSON values (`equals`,
  `beginsWith`, `exists`, alternatives with `||`).
- `dnt`: whole sheets or columns; system columns (`:translate`, `:uid`,
  `:rollout`, `:regional`) are excluded automatically.

Ready-made configs per strategy and vendor: aem.live
`/docs/ew/administering/translation-config-samples`.

**Config gotchas** (checked against the `da-nx` loc source and a working
Google setup):
- **The provider is one config value per site**, not a UI choice. The value
  is lowercased with spaces → `-` and must equal a connector folder in
  `adobe/da-nx`; empty = Google.
- `source.language` takes the language **name**, matched exactly and
  case-sensitively against a `languages` row's `name` (`English`, not `en`).
- Conflict-behavior values are split on `,` and trimmed; the first item is
  the default. **Literal quotes in the cell break it silently**: the
  options become `"merge` / `overwrite"`, and since the code tests
  `behavior === 'overwrite'`, every save merges.
- `complete.aem.preview` / `complete.aem.publish` are documented, but no
  code in current `da-nx` `main` reads them (they only appear in GLaaS test
  fixtures). Preview translated pages manually.

Minimal working config (Google, pages under `/en/`):

| `config` key | value |
|---|---|
| `translation.service.name` | `Google` |
| `translation.service.all.env` | `prod` |
| `source.language` | `English` |
| `sync./translate./copy./rollout.conflict.behavior` (four rows) | `overwrite, merge` |

| `languages`: name | code | location | actions |
|---|---|---|---|
| English | en | /en | Skip, Translate |
| French | fr | /fr | Skip, Translate |

Optional `dnt-content-rules` tab (column `content`) with a test phrase to
check do-not-translate survives.

## 4. Other translation vendors

Built-in connectors besides Google: **Lionbridge, Smartling, Trados**
(account + credentials in the config). Adobe builds new connectors on
request, only if the vendor offers: a **public cloud HTTPS REST API** (no
on-prem, VPN-only or IP-allowlisted hosts), **non-interactive auth** (OAuth
client credentials or API key; no per-user SSO), standard locale codes,
burst-tolerant rate limits (retries on 429/5xx, five parallel downloads),
**HTML in and out that preserves markup and honours `translate="no"`**, and
stable IDs per file. Connector calls run from the browser (proxied when the
vendor lacks CORS). Own connectors: implement `isConnected`, `connect`,
`sendAllLanguages`, `getStatusAll`, `saveItems` (optional
`cancelTranslation`, `serviceOptions`) and contribute to `adobe/da-nx`
(`nx/blocks/loc/connectors`). The real contract and plumbing (mandatory
`dnt` export, private proxy repo, merge-is-release delivery) are in
[[ew-translation-connector-internals]].

Outside DA (SharePoint/Google Drive sources), aem.live points to the
built-in machine translation in Office/Google Workspace and to their APIs
for bulk vendor workflows.

## 5. UI strings: placeholders

Strings shown by code ("Read more", "Search") belong in a `placeholders`
sheet per language folder, never in JS/CSS.

```js
import { fetchPlaceholders } from '/scripts/placeholders.js';
const { readMore } = await fetchPlaceholders('fr'); // /fr/placeholders.json
```

Keys are camel-cased (`about us` → `aboutUs`, `VIP lounge` → `VipLounge`);
no argument = root sheet. **`scripts/placeholders.js` is no longer in the
boilerplate**: copy it from `adobe/aem-block-collection` (`scripts/`).

## 6. Code a boilerplate project still needs

The Translate app produces folders of content; making the site behave
multilingually is project code. Gaps in a stock `aem-boilerplate` 1.3.0:

- `scripts.js` sets **`document.documentElement.lang = 'en'`** hard-coded:
  derive it from the first path segment or page metadata.
- Header/footer load one `nav`/`footer`: resolve `/<lang>/[<market>/]nav`
  and `footer` instead (market copy first, then language).
- **Link rewriting** for market visitors: keep an index of localized pages
  per market, and rewrite `href`s (or handle `click`) so `/en/brands` links go
  to `/en/uk/…` when that version exists; same for `fetch`es of fragments.
- **`hreflang` sitemap** via the sitemap config in the Configuration
  Service, e.g. `/en/` ↔ `hreflang="en-GB"` `/en/uk/`.
- Language/market switcher persisting the choice (`localStorage` or a
  cookie).

## 7. What this means for AEM

On the Java stack translation is language copies + translation projects +
MSM live copies inside the JCR, driven by `translation_rules.xml` and
connectors (see `translation-i18n/`). On EDS the same jobs are split: the
DA Translate app does translation projects and rollout (merge ≈ live-copy
inheritance with local edits kept), its `.da/translate` sheet replaces
translation rules, and the **Multi-Site Manager** in DA covers base/satellite
site inheritance. Language detection, link rewriting and hreflang are
project code and config, not product features.

## References
- [Translation and Localization](https://www.aem.live/docs/translation-and-localization) (strategy, detection, link rewriting, hreflang)
- [Strategies for translation, localization, and rollout](https://www.aem.live/docs/ew/administering/translation-strategy)
- [Translation projects](https://www.aem.live/docs/ew/authoring/translation-projects)
- [Set up translation](https://www.aem.live/docs/ew/administering/set-up-translation), [config options](https://www.aem.live/docs/ew/administering/translation-config), [config samples](https://www.aem.live/docs/ew/administering/translation-config-samples)
- [Connecting to a translation service](https://www.aem.live/docs/ew/administering/translation-service-connection)
- [Placeholders (developer)](https://www.aem.live/developer/placeholders)
- [Multi-Site Manager](https://www.aem.live/docs/ew/administering/multi-site-manager)
