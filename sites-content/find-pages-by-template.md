# Finding which pages use a template (and the broken Template search facet)

## The methods, ranked

1. **Templates console References rail — the canonical UI method, zero setup.**
   Tools → Templates (`/libs/wcm/core/content/sites/templates.html/conf/...`), check the template, open left rail → **References** → lists the pages created from it. Works on prod with a plain author login; no CRX, no configuration. Under the hood it runs the same `jcr:content/cq:template` lookup via reference search, so on very large trees it can be slow or cap the displayed list.

2. **QueryBuilder JSON — complete result set, scriptable, works wherever the servlet isn't blocked.**
   Every page stores its source template in `jcr:content/cq:template`:
   ```
   /bin/querybuilder.json?path=/content&type=cq:Page
     &property=jcr:content/cq:template
     &property.value=/conf/<app>/settings/wcm/templates/<template-name>
     &p.limit=-1&orderby=path
   ```
   Paste in a browser while logged into author (URL-encode `:` and `/` in param names if curling). **Trap:** `p.hits=selective&p.properties=jcr:path` silently returned `total: 0` on the 6.5 LTS instance — use plain hits and read `hits[].path`.
   JCR-SQL2 equivalent: `SELECT * FROM [cq:PageContent] AS p WHERE p.[cq:template] = '<template path>'` (hits are `jcr:content` nodes; parent = page).

3. **Sites search rail Template facet — only if the search form includes it, and see the bug below.**
   Sites/XF console → left rail → Filter (alt+4) → jumps into omnisearch with predicate facets. Which facets exist is **content config, not product**: Tools → General → **Search Forms** → "Sites Admin Search Rail" → drag **Templates Predicate** in, set required Property Name = `jcr:content/cq:template`. Saving writes `/conf/global/settings/cq/search/facets/sites` (overlays the `/libs/settings/cq/search/facets/sites` default; delete the `/conf/global` node to revert). Runtime-authorable — no deployment — but can be shipped in `ui.content` for config-as-code.

## Gotchas verified on AEM 6.5 LTS local author (Aug 2026)

- **Search Forms editor's Done button can be permanently disabled** before you change anything: the stock Sites rail form ships an invalid field (LiveCopy Status predicate with an empty required `name`), and the wizard gates Done on whole-form validity. Workaround: fix/fill that predicate's property name, or submit the builder form directly (`form#builder-form` POSTs to `<facet-path>/_jcr_content`; all sidecar inputs are already in the DOM).

- **The Template facet's selection is broken by an unescaped CSS selector in Granite JS.** Server side is fine (suggestion servlet `/mnt/overlay/cq/gui/content/coral/common/form/templatefield/suggestion.0.10.html?query=...` returns clean `<coral-buttonlist>` markup with template paths as button values), but on picking a suggestion or confirming the picker dialog, the autocomplete's duplicate-check runs
  `querySelector("coral-tag input[type=hidden][value='<item display text>']")`
  with the item's **display text embedded unescaped** — and picker-card text contains newlines from markup → `Uncaught SyntaxError: ... is not a valid selector` → tag creation aborts → the input silently clears and no filter is applied. Affects every selection path (typed suggestion, picker Select). Symptom signature: field empties on click, no coral-tag appears, SyntaxError in console. Not caused by custom config — same code path serves a stock Template facet. Presumably fixed in some Granite service pack; test with one click before relying on it on any given instance.

- **Automation note:** the omnisearch overlay aggressively steals keyboard focus back to its main fulltext box — synthetic/scripted typing into rail predicate fields lands in the wrong input. Interact via the picker dialog or real mouse events when driving this UI with tooling.

## Related

- XF templates' "pages" live under `/content/experience-fragments/**` — they're `cq:Page` nodes too, so all methods above cover them (`path=/content` already includes them).
- Template *usage* ≠ template *reference*: a page embedding an XF (e.g. `fragmentVariationPath` in initial content) references the fragment's content path, not the template — search on the fragment path for "where is this rendered".
- See [wcm-reference-provider-contract.md](wcm-reference-provider-contract.md) for how the reference aggregation behind publish-wizard/References works.