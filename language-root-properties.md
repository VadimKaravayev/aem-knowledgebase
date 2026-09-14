# Language-root properties: `jcr:language` / `cq:isLanguageRoot`

How AEM recognizes a folder or page as a *language root* (`/content/<site>/fr`,
`/content/dam/<site>/fr`, `/content/experience-fragments/<site>/fr`), and what to stamp when
creating such roots programmatically.

## Detection: two mechanisms

`LanguageManager` (and everything built on it — Sites/XF "Language Copies" rails, Assets
multilingual views, translation-project UIs) recognizes a language root by **either**:

1. **Path heuristic** — the segment name is a recognizable locale code (`fr`, `de`, `de_de`,
   `zh-cn`). Works with no properties at all.
2. **Explicit marker** — `cq:isLanguageRoot = true` on the node (folder, or page `jcr:content`).
   The only way for non-ISO segment names (`latam`, brand-specific folders).

Original/source roots often have *neither* (stock WKND `/content/experience-fragments/wknd/language-masters/en`
is a bare `sling:OrderedFolder` — detection rides the path heuristic). Targets created by AEM's
own language-copy tooling get the full property set.

## What AEM's own tooling stamps on a created language root

Observed on a tooling-created `.../language-masters/fr` (verified July 2026, local SDK):

```
jcr:language          fr                      ← locale of the tree
cq:isLanguageRoot     true                    ← explicit root marker
jcr:title             French                  ← display name (consoles show this, not "fr")
cq:conf               /conf/<site>            ← config context: allowed templates/policies below
cq:cloudserviceconfigs [/libs/settings/cloudconfigs/translation/...]   ← see trap below
```

## Recipe for custom connectors creating language roots

Stamp `jcr:language`, `cq:isLanguageRoot`, `jcr:title` (e.g. `Locale.forLanguageTag(...)
.getDisplayName(Locale.ENGLISH)`), and copy `cq:conf` from the source-language sibling —
without `cq:conf`, authors may be unable to create XFs/pages under the new folder (no allowed
templates). Detecting "this created ancestor IS the language root": it's the level where the
segment name differs between source and target path (`en` → `de`); deeper ancestors copy
names unchanged.

**Trap — do NOT copy `cq:cloudserviceconfigs`**: it binds the tree to AEM's built-in
translation integration (`default_translation`). A third-party connector that copies it
invites the AEM translation framework to claim the tree it just created.

Missing properties degrade, not break, for ISO-named folders: rails and filters still work via
the path heuristic; the visible symptom is consoles showing raw segment names (`de`) instead
of display names. For non-ISO names, detection fails entirely without `cq:isLanguageRoot` —
and note that path-string utilities that never read the repo (e.g. `LanguageUtil.getLanguageRoot`
on a plain string, or custom regex fallbacks) cannot be rescued by the marker.

Pages need the same treatment: a Sites language root is a `cq:Page` whose **`jcr:content`**
carries `jcr:language` + `cq:isLanguageRoot`. Two extra gotchas there: stamp AFTER the copy
(a shallow `PageManager.copy` brings the source's jcr:content, which lacks the props), and
RE-stamp after any refresh that replaces `jcr:content` wholesale — the replacement silently
wipes previously stamped (or hand-added) props.

Implemented in the Crowdin connector's `LanguageCopyServiceImpl` (folder branch of
`recreateAncestorStructure` + page stamping in the copy/refresh paths), July 2026.