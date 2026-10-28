# Experience Workspace translation connectors: three-repo architecture, loading, delivery, contract gaps

How a translation vendor connector for the Experience Workspace (DA) Translate
app is actually built, loaded and shipped, as found in the public source
(`adobe/da-nx`, `adobe/da-live`, `da-sites/da-etc`) and Adobe's own
connector-building agent skill. Goes beyond the aem.live "Connecting to a
translation service" page, which documents the interface but not the
plumbing. Config-level usage is in [[edge-delivery-translation-localization]].

---

## 1. A connector spans three repos plus docs

| Piece | Repo | Role | Public |
|---|---|---|---|
| Connector | `adobe/da-nx` → `nx/blocks/loc/connectors/<name>/` | Browser module: send, poll, save | yes |
| Credential endpoint | `da-sites/da-etc` → `src/routes/ints.js` | Cloudflare Worker; reads the site's `.da/translate` (+ keys sheet) server-side, exchanges or validates the vendor credential | yes |
| Proxy service | `adobe/da-translate` → `src/services/<name>.js` | Forwards browser calls to the vendor, injects credentials, solves CORS | **private** |
| Docs | aem.live (`docs/ew/administering/…`) | Supported list, config keys, samples | Adobe publishes |

- da-etc keeps a hardcoded registry `TOKEN_FETCHERS = { trados, lionbridge,
  smartling, globallink, deepl, lilt, … }`; static-key vendors are also added
  to `STATIC_KEY_SERVICES`, which exposes only a boolean `/status` action.
- **New connectors must route every call through da-translate.** The
  direct-to-vendor pattern of Lionbridge, Trados and Smartling predates the
  proxy and Adobe's skill says not to copy it. An external contributor cannot
  write the private proxy piece, so a vendor-written connector still needs
  Adobe to add one file plus routing there.
- Two auth archetypes: **A** token exchange (OAuth/client credentials,
  GlobalLink pattern: short-lived token in `sessionStorage`) and **B** static
  API key (Lilt/Translated pattern: key never reaches the browser; the proxy
  resolves it from da-etc). The vendor credential never goes in
  `Authorization`, which is reserved for the IMS token; use
  `x-<name>-authorization` / `x-<name>-origin`. Per-tenant vendor hosts must
  be validated against an env allowlist in the proxy.

## 2. How the Translate app finds a connector

- `translation.service.name` (default `Google`) is lowercased, spaces become
  `-`, and the app does
  `import(\`../../connectors/${serviceName}/index.js\`)`
  (`views/translate/index.js`, `views/options/options.js`). **No registry,
  no runtime injection**: the config value must equal a folder name inside
  Adobe's repo.
- One provider per site config; there is no provider picker in the UI.
- **No fork route.** `da-live/scripts/utils.js` `getNx()` builds
  `https://${branch}--da-nx--adobe.aem.live`; only the branch is variable,
  the org and repo are hardcoded.

## 3. Contract gaps between the docs and the code

- **`export const dnt = { addDnt }` is mandatory** but undocumented. The host
  calls `connector.dnt.addDnt(...)` directly (`views/translate/index.js`,
  `project/views/translate.js`). A connector written strictly to the aem.live
  page breaks.
- `connectors/sample/index.js` is a 21-line empty stub (not even
  `isConnected`), despite the docs calling it a starting template. Use a
  recent proxy-based connector (Translated, Lilt) as the model.
- `addDnt` marks protected content three ways: `translate="no"` on elements,
  `<span translate="no" class="dnt-text">` around protected strings, and on
  table rows/cells. JSON sheets are converted to HTML with custom attributes
  (`data-type`, `key`, `sheet-attrs`). The vendor must round-trip all of it
  unchanged.
- Status vocabulary is `created / in progress / translated / error /
  cancelled`. Only double-L `cancelled` is terminal (Lionbridge's
  `canceled` is a known wart). `sendAllLanguages` must return right after
  upload with `created`; polling happens only on the author's **Get status**
  click (`getStatusAll`), never automatically.
- Shared helpers are mandatory in review: `downloadQueue` (max five
  concurrent), `fetchWithRetry`, `utils/auth.js`. Static-key support
  (`checkConnection`, static `serviceOptions`) was introduced by connector
  PRs rather than on its own.

## 4. Delivery: merge is release

- da-nx has no deploy workflow: it is an Edge Delivery site synced by AEM
  Code Sync. Production da.live loads it from `/nx` (`setNx('/nx')` in
  da-live `scripts/scripts.js`), served from da-nx `main`. **Merged to `main`
  = live for every customer**; there is no versioning or install step.
- da-etc deploys to Cloudflare on push to `main` (GitHub Action + wrangler).
  Rollout order for a new vendor: da-etc, then da-translate, then da-nx.
- Customer side is config only: `translation.service.name` plus the vendor
  keys (secrets in the separate keys sheet).
- **Previews before merge:** every da-nx branch is live at
  `https://<branch>--da-nx--adobe.aem.live`, and `?nx=<branch>` loads it
  inside real da.live (the param is honored on production hosts too).
  `?nx=local` loads `http://localhost:6456` (`npm run local` = `aem up
  --port=6456`), so a connector can be prototyped against a real site without
  any merge.

## 5. Adobe's connector skill and contribution reality

- Adobe wrote an agent skill `build-translation-connector` (SKILL.md, seven
  references, eleven templates, `scripts/check-connector.mjs` static checker)
  that documents the conventions reviewers enforce. Its PR (da-nx #875) was
  closed unmerged with "until we move loc to a new repo"; the content is
  still retrievable from the PR's head commit
  (`git fetch origin pull/875/head`). Expect the loc code to change repo.
- Every shipped connector was written by Adobe engineers. An external
  Lionbridge PR was closed and replaced by Adobe's own; the external DeepL PR
  (#746) stalled in review on internal conventions. Adobe's own connector
  PRs merged in roughly three weeks each.
- Adobe's documented line: engineering builds connectors on request (contact
  your Adobe representative with the vendor name and API docs link);
  contributing your own is the exception.

## 6. What this means for AEM

Nothing carries over in code from Java-stack AEM translation connectors (no
JCR, language copies, `translation_rules.xml` or MSM). The domain knowledge
does: segmentation, do-not-translate handling, locale mapping and status
modeling. A vendor targeting both stacks ships two unrelated integrations,
and on Experience Workspace the release gate is Adobe's merge, not the
vendor.

## References
- [Connecting to a translation service](https://www.aem.live/docs/ew/administering/translation-service-connection)
- [Translation config options](https://www.aem.live/docs/ew/administering/translation-config)
- `github.com/adobe/da-nx` (`nx/blocks/loc/`), `github.com/adobe/da-live`, `github.com/da-sites/da-etc`
