# Where a Translated i18n Dictionary Can Be Written (`/content/cq:i18n`)

Why a translation connector cannot write a translated dictionary next to its source, and the path it must use instead.

The short version: **an i18n dictionary's translation does NOT go beside its source. `/apps` and `/libs` are immutable at runtime on AEMaaCS, so the only writable home for a dictionary is `/content/cq:i18n`. AEM's own runtime translation writes to `/content/cq:i18n/<projectName>/<lang>.json` regardless of whether the source sits in `/apps` or `/content`.**

This makes i18n the odd one out among translatable content types. A page, an experience fragment, a content fragment and an asset all get a target that mirrors the source with the language swapped (`/content/site/en/x` to `/content/site/de/x`). A dictionary does not.

---

## Why the obvious target is wrong

The intuitive mapping for `/apps/myproject/i18n/en.json` is `/apps/myproject/i18n/de.json`. On AEMaaCS that write can never succeed.

Adobe, on the mutable/immutable split:

> The `/apps` and `/libs` areas of AEM are immutable, and after AEM starts, you cannot create, update, or delete content in these areas at runtime. Any attempt to change an immutable area at runtime fails.

> Everything else in the repository, `/content`, `/conf`, `/var`, `/etc`, `/oak:index`, `/system`, `/tmp`, and so on, are all mutable areas.

And specifically about dictionaries, from the AEMaaCS Translator page:

> no runtime editing or translation of dictionaries possible, as `/apps` is immutable in AEM as a Cloud Service environments

The failure mode is worth knowing: this is not caught at build time. It surfaces as a runtime write failure on the import leg of a translation pipeline, after the translation has already come back from the vendor.

## Where it goes instead

From the same Adobe page:

> the new AEM runtime translation process for i18n dictionaries will create translated dictionaries in `/content/cq:i18n/<projectName>`, regardless if the source dictionary is stored in `/apps` or `/content`.

Note **"regardless"**. The source stays where it is and is only ever read. You do not need to relocate a `/apps` dictionary to translate it, and you should not try to.

This works at lookup time because Sling merges every dictionary it finds in the repository into one `ResourceBundle`. A German dictionary at `/content/cq:i18n/myproject/de.json` supplies strings for an English source that shipped in `/apps`, with `/apps` untouched.

## The two node shapes

Both are valid and both appear in a stock repository.

**JSON file** (preferred, and what the runtime translation process produces):

```
/content/cq:i18n/<projectName>   [sling:Folder]
    de.json                      [nt:file] [mix:language]
      + jcr:language = de
      jcr:content                [nt:resource]
        + jcr:mimeType = application/json
```

**Message nodes** (the older form). Sling accepts a child of `jcr:primaryType` `sling:MessageEntry` **or a child of any type carrying the `sling:Message` mixin**. The Translator UI writes the second kind, as `nt:folder` nodes, and its node names are lowercased while `sling:key` keeps the original case:

```
/apps/<path>/i18n/en             [sling:Folder] [mix:language]      ← code
    searchButtonText             [sling:MessageEntry]
      + sling:key = searchButtonText, sling:message = Search

/content/cq:i18n/<path>/i18n/ar  [sling:Folder] [mix:language]      ← Translator UI
    searchbuttontext             [nt:folder] [sling:Message]
      + sling:key = searchButtonText, sling:message = ...
```

A reader that only accepts `sling:MessageEntry` misses every entry the Translator has saved. `sling:key` is optional; without it the node name is the key.

Apache Sling recommends the JSON form on performance grounds: since i18n bundle 2.4.2, JSON dictionaries load much faster than `sling:MessageEntry` ones and should be preferred.

The `mix:language` mixin plus `jcr:language` is what makes a node a dictionary. For a JSON dictionary both go on the **`nt:file` node itself**, not on `jcr:content`. In FileVault that means a `<name>.json.dir/.content.xml` beside the file. See [[htl-i18n-page-language-gotcha]] for that packaging recipe.

## The path rule

Adobe writes `<projectName>` as a single segment, but it is really the source's folder path and can be deeper. **Adobe's own two tools disagree on the exact shape.** Both observed on a stock AEMaaCS SDK instance:

| Tool | Source folder | Target |
|---|---|---|
| Translation projects (`translation-job-service`) | `/libs/wcm/core/i18n` | `/content/cq:i18n/wcm/core/zh.json`, **`/i18n` dropped** |
| Translator UI | `/apps/foundation/components/search/i18n` | `/content/cq:i18n/foundation/components/search/i18n/ar`, **path mirrored** |

Lookup does not care (see precedence below: Sling finds dictionaries by query, not by path), so either works. For a connector, **mirror the whole source folder path**: it cannot collide (dropping `/i18n` maps `/apps/a/i18n` and a hypothetical `/apps/a` dictionary to the same place) and it points visibly back to its source.

| Source folder | Target for `de` |
|---|---|
| `/apps/myproject/i18n` | `/content/cq:i18n/myproject/i18n/de.json` |
| `/content/cq:i18n/…/i18n` (already mutable) | a sibling: `…/i18n/de.json` or `…/i18n/de` |
| `/libs/wcm/core/i18n` | the same mirror: `/content/cq:i18n/wcm/core/i18n/de.json`. Translation projects offer `/libs` dictionaries too; the source is only read, so "do not modify `/libs`" does not apply. Adobe ships many languages there itself, and `/libs` beats `/content`, so only languages Adobe does not ship take effect for its keys |

## Version differences, and why the rule still holds on 6.5

The two Adobe pages say different things, and the difference is real:

- **AEMaaCS**: `/apps` immutable, runtime translation writes to `/content/cq:i18n`. Translator at `/libs/cq/i18n/gui/translator.html`.
- **AEM 6.5**: the Translator page says dictionaries are created at `/apps/<projectName>/i18n` and instructs *"Only edit dictionaries that are created for your project and reside under `/apps`."* It does not mention `/content/cq:i18n` at all. It also warns that the Translator *"will only save translations for languages that are actually present underneath the path"*, so a target language with no existing node is silently dropped.

On 6.5 `/apps` is writable at runtime, so the naive target does not hard-fail there. It is still the wrong choice: a code deployment overwrites `/apps`, so translations written there are destroyed on the next release.

**Write to `/content/cq:i18n` on both.** It is mandatory on AEMaaCS, survives deployments on 6.5, and matches what AEM's own translation service does.

## Precedence: `/apps` shadows the translation

Sling finds dictionaries by query on `mix:language` + `jcr:language`, anywhere. For one language, entries are ordered like the resource resolver search path: **`/apps` first, then `/libs`, then everything else**. So a key that `/apps` already defines for German keeps its `/apps` text even after a translation lands in `/content/cq:i18n`. Adobe: *"dictionaries with identical key/value pairs must not exist simultaneously in `/apps` and `/content/cq:i18n`, as the dictionary in `/content/cq:i18n` will never be used for rendering."*

The common trap is a project that ships `/apps/myproject/i18n/de.json` as code and then translates the same folder into German: every key already in the shipped file is silently ignored. Check for a same-language sibling under `/apps` before sending, and warn. New keys still work, so it is a warning, not a blocker.

Locale fallback is `de-DE` → `de` → default (`en`).

## Where the source text comes from

- **Often there is no source-language dictionary.** The Granite convention makes the key the English text (`/apps/myproject/i18n` holds `de.json` and `fr.json`, no `en.json`). Translation projects cover this with a source option **"English (en), From dictionary keys"**. A connector needs the same fallback, and only for an English source.
- **Send `sling:message`, not the key.** For a key like `searchButtonText` the text to translate is the message, "Search".
- **The out-of-the-box XLIFF export gets this wrong.** `GET <folder>/<lang>.dict.xliff` (`com.adobe.granite.i18n.impl.DictImportExportServlet`) is XLIFF 1.1 with the **key** in `<source>`, the message in `<target>`, and a fake `original="/libs/cq/i18n/<lang>"`; its import (`POST <folder>.dict.xliff`) writes `sling:Message` nodes wherever it is posted, `/apps` included. Fine for a human in the Translator, not a base for a connector.
- **Listing dictionaries has no public API either.** The Translator uses `/libs/cq/i18n/translator.i18npaths.json` (every folder) and `/bin/translation/i18n.json?path=<folder>` (its languages), both internal. Their logic is one query, `//element(*, mix:language)[@jcr:language] order by jcr:path`, grouped by parent folder; `mix:language` is in the node-type index, so it is fast. Run that query yourself.

## Consequences for a connector

- **Do not derive the i18n target from the language-copy logic** used for pages and assets. It needs its own branch. A connector that computes one target path for all content types will produce an unwritable path for every dictionary.
- **A classification rule that only looks at `/apps` and `/etc` misses real dictionaries.** Authorable ones live in `/content/cq:i18n`, which is exactly where the Translator puts them. A content picker built on such a rule will not list them.
- `/etc` is mutable, so a dictionary there is writable, but `/etc` is deprecated for new content under repository modernization. Prefer `/content/cq:i18n`.
- Source and target node types can legitimately differ, for example a JSON file source and a `sling:Message` target the Translator made. **Write into an existing target in its own format**; for a new target, take the source's format (JSON for a key-only source).
- **Merge, never overwrite.** A target may hold keys the source does not, added by hand in the Translator. Write only the source's keys, keep the rest, and copy `sling:basename` when the source has one, or basename-scoped lookups will not see the translation.
- **Do not hide `/apps` dictionaries on AEMaaCS.** One connector does, because `/apps` is not writable; that makes the most common dictionaries untranslatable. The source only has to be readable; the target is `/content/cq:i18n`.
- Authors often cannot write `/content/cq:i18n`. A connector that writes with a service user should check the author's own write permission on the target path at submit.

---

## Sources

- [Internationalizing UI Strings, AEMaaCS](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/configuring-and-extending/internationalization/translator) — the `/content/cq:i18n/<projectName>` rule and the `/apps` immutability statement.
- [What is Mutable and Immutable content in AEM as a Cloud Service?](https://experienceleague.adobe.com/en/docs/experience-manager-learn/cloud-service/developing/basics/mutable-immutable)
- [AEM Project Structure, AEMaaCS](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/developing/aem-project-content-package-structure) — code/content package separation.
- [Using Translator to Manage Dictionaries, AEM 6.5](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/implementing/developing/components/internationalization/i18n-translator)
- [Internationalizing Components, AEM 6.5](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/implementing/developing/components/internationalization/i18n)
- [Apache Sling, Internationalization Support (i18n)](https://sling.apache.org/documentation/bundles/internationalization-support-i18n.html) — JSON dictionaries preferred since 2.4.2; the `sling:Message` mixin; search-path precedence.
- [Managing Translation Projects, AEMaaCS](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/sites/administering/reusing-content/translation/managing-projects) — dictionaries added to a translation job get language copies in `/content/cq:i18n`.

Amended 2026-09-25 on the same SDK author: the two Adobe path conventions, the `nt:folder` + `sling:Message` shape the Translator writes, the Translator's endpoints and XLIFF export read directly (`DictImportExportServlet`, `I18nDictionaryServlet` in `cq-wcm-translation`), and 38 dictionaries listed by the `mix:language` query.

Docs read September 2026. Repository shapes verified the same day on a local **AEM as a Cloud Service SDK** author (`cq-wcm-core 5.16.4`, run mode `sdk`): `/content/cq:i18n/wcm/core/zh.json` written by `translation-job-service`, and `/content/cq:i18n/foundation/components/search/i18n/ar` as a `sling:Folder` with `mix:language`. The 6.5 statements above are from Adobe's documentation and were **not** reproduced on a 6.5 instance.