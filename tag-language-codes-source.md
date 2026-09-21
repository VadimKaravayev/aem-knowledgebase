# Where the tag "Languages" picker gets its language list

The multi-select at the top of Tag Edit (Tools → Tagging → edit a tag) is rendered by
`/libs/cq/tagging/gui/components/tagedit/languagepicker/render.jsp`. It draws from **two
different sources depending on a feature toggle**, and neither of them is JDK locale data.

## Source 1 (legacy branch): a plain property on the tag root

With toggle `FT_CQ-4363942` **off**, the option list is `TagManager.getSupportedLanguageCodes()`.
The javadoc says only *"the List of language codes support by Tag"*. Disassembling
`com.day.cq.tagging.impl.JcrTagManagerImpl` (bundle `com.day.cq.cq-tagging` 5.13.38) shows the
whole method body is:

```java
resourceResolver.resolve(tagBasePath)
                .adaptTo(ValueMap.class)
                .get("languages", new String[0]);   // wrapped in Arrays.asList
```

So it is nothing but the multi-value `languages` String[] property on the tag base path,
i.e. `/content/cq:tags`. No computation, no fallback, no config. Stock value on a fresh instance:

```
en, de, es, fr, it, pt_br, zh_cn, zh_tw, ja, ko_kr, uk
```

Adding a language to the authoring UI is therefore a one-property content change on
`/content/cq:tags`, not an OSGi config and not a code change.

## Source 2 (new branch): the shared languages resource

With `FT_CQ-4363942` **on**, the tag root property is ignored entirely and the picker lists every
child of `/mnt/overlay/wcm/core/resources/languages`, which is `/libs/wcm/core/resources/languages`
(173 nodes on a 2026 Cloud SDK) plus any `/apps/wcm/core/resources/languages` overlay. That is the
same table Sites uses, so on a toggled instance the tag picker offers every locale AEM knows about.

The toggle state is visible at `/etc.clientlibs/toggles.json` (`enabled` array). It was off on a
local Cloud SDK in Sept 2026, so the legacy branch is still what most people see.

## Labels come from the languages resource either way

For each code the JSP reads `/mnt/overlay/wcm/core/resources/languages/<code>`:

```json
{"language": "German", "country": "Switzerland"}    // de_ch → "German (Switzerland)"
{"language": "Arabic", "country": "*"}              // ar    → "Arabic" (the "*" suppresses it)
```

Both values go through `i18n.getVar(...)`, so the label is localized for the author. Codes are
normalized `-` → `_` and lowercased before the lookup, so `zh-CN` and `zh_cn` resolve to the same
node.

**Fallback:** a code with no node under `languages/` is dropped from the list *unless* the tag being
edited already has a localized title for that locale (`origTag.getLocalizedTitle(locale)`), in which
case the option survives with the raw code as its label. This is why a tag can display a language
that the picker does not otherwise offer.

## What this means for a translation connector

- Localized tag titles live on the tag node as `jcr:title.<code>`, keyed by exactly these
  lowercase-underscore codes (`pt_br`, not `pt-BR`). Serialize and write back using that spelling.
- Do not derive the offered set from `Locale.getAvailableLocales()` or from the languages resource
  when the instance is on the legacy branch. Read `/content/cq:tags@languages` via
  `TagManager.getSupportedLanguageCodes()`, which is the same thing and survives the toggle flip.
- The offered set is not the same as the present set. A tag may carry `jcr:title.<code>` for a code
  that is not in the array (imported content, a since-removed language), so iterate the tag's actual
  properties when serializing, and use the array only to decide what to *offer*.
- Labels are content, not code. If a customer needs a language name shown differently, that is an
  overlay at `/apps/wcm/core/resources/languages/<code>`, not a connector change.

## Verify in 30 seconds

```bash
curl -s -u admin:admin "http://localhost:4502/content/cq%3Atags.json" | jq .languages
curl -s -u admin:admin "http://localhost:4502/mnt/overlay/wcm/core/resources/languages/de_ch.json"
curl -s -u admin:admin "http://localhost:4502/etc.clientlibs/toggles.json" | grep -o "FT_CQ-4363942"
```

Verified Sept 2026 on a local AEM Cloud SDK author (`cq-tagging` 5.13.38) by disassembling the
bundle and reading the JSP out of the repository with
`curl .../crx/server/crx.default/jcr:root/<path>/jcr:content/jcr:data`.

**6.5:** `getSupportedLanguageCodes()` disassembles to the identical body in `cq-tagging`
5.12.7.CQ650-B0004 (6.5 quickstart on disk), so source 1 is the whole story there. The toggle API
`com.adobe.granite.toggle.api.ToggleRouter` that the new branch imports is absent from uber-jar
6.5.22 and 6.5.23, so treat the languages-resource branch as Cloud-era only.
