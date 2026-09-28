# AEM `translation_rules.xml` — translatability filter **and** declarative reference-finder

How AEM's translation framework decides what content goes out for translation, and the under-appreciated second job of the same file: telling the connector which properties are **references** to other content (assets, fragments) that need their own language copies — a declarative alternative to hand-coded reference finders.

The short version: **`translation_rules.xml` is config-as-code that maps JCR nodes/properties to translatability verdicts; its `<assetNode>` entries additionally form an author-owned registry of "which property on which component is an asset/fragment reference," replacing custom Java reference-discovery code for the common case.**

---

## Where the file lives (priority order — first found wins)

1. `/conf/global/settings/translation/rules/translation_rules.xml`  *(recommended / project)*
2. `/apps/settings/translation/rules/translation_rules.xml`
3. `/etc/workflow/models/translation/translation_rules.xml`  *(legacy)*
4. `/libs/settings/translation/rules/translation_rules.xml`  *(AEM OOTB default)*

Stored as a JCR `nt:file` (`jcr:content/jcr:data` binary). It is the format AEM's own "Translation Configuration" UI generates, so existing files are portable across connectors. The three custom AEM translation connectors examined (translated.com, blackbird, LanguageWire) all parse this same schema with their own engines and **none** use `com.adobe.granite.translation.*` / `TranslationManager`.

---

## Schema / rule taxonomy

Root is `<nodelist>`. Two scopes: rules **inside** a `<node path>` context (longest path wins), and **top-level** entries (`<assetNode>`, `<contentFilterNode>`) that apply at content-item scope.

| Element | Attributes | Meaning |
|---|---|---|
| `<node path="…">` | `path` | Context scope. Adobe's precedence rule is *"when multiple rules target the same node, the rule lower in the file is used"* — and since the Translation Rules editor always writes contexts general-path-first / specific-path-last, **"lower in the file" == "longest matching path"**. So sorting contexts longest-path-first is a faithful implementation |
| `<property>` | `name`, `translate`, `inherit`, `updateDestinationLanguage` | Translate (or not) a property **by name**. `name` may be a **relative path** (`image/alt` = property `alt` on child node `image`). `translate` = send the value for translation. **`inherit`** (Adobe docs, default `true`) = whether the property rule is **inherited by child nodes**; `false` → "applied only to that specific node" — a rule-**scope**/extraction concern, *not* write-back. `updateDestinationLanguage` = for language-code props (e.g. `jcr:language`): set the target to the destination locale instead of sending for translation (write-back) |
| `<node pathContains="…"><property/></node>` | `pathContains` + nested `name`/`translate` | Property rule scoped to nodes whose **path contains** a substring (e.g. exclude `text` under `/cq:annotations`) |
| `<node resourceType="…"><property/></node>` | `resourceType` + nested props | Property rule scoped to a component `sling:resourceType` (matched on the nearest ancestor carrying it; exact string, no resource-super-type inheritance) |
| `<filter><node containsProperty=… propertyValue=… isDeep=…/></filter>` | `containsProperty`, `propertyValue`, `isDeep` | **Node**-level skip: a node carrying `prop==value` is excluded. `isDeep=true` (**Adobe default** when the attribute is absent) → walk ancestors + prune the whole subtree (`NON_TRANSLATABLE`); `isDeep=false` → skip the node's own props only, keep descending (`ONLY_CHILDREN_TRANSLATABLE`) |
| `<assetNode/>` | `assetReferenceAttribute`, `resourceType`, `checkInChildNodes`, `createLangCopy` | **Reference discovery** — see below |
| `<contentFilterNode/>` | `key`, `value`, `createLanguageCopy` | Tag-based gate on whether a whole content item is translated / language-copied (e.g. `cq:tags=do-not-translate` → skip) |

**Defaults when no rule matches:** a property → **excluded** (opt-in: you name what's translatable); a node → **translatable** (a node is just a container the walk must descend through to reach opted-in properties — pruning is done only by explicit `<filter>`).

**Walk shape:** recurse the node tree, ask the rules per node then per property, and **skip `cq:Page` children** (so one page's extraction doesn't bleed into nested pages).

---

## The `<assetNode>` insight — declarative reference discovery

A page's translatable content isn't all inline: components reference other content by path —
`fileReference=/content/dam/…` (image/asset), `fragmentPath=/content/experience-fragments/…` (XF),
`fileReference=…` to a content fragment. These reference **values are paths, not text** (never send a
path to a translator), but the *target* often needs translating, and the translated page must point at
the **translated copy** of the target.

`<assetNode>` declares exactly that:

```xml
<assetNode assetReferenceAttribute="fileReference" resourceType="foundation/components/image"
           checkInChildNodes="false" createLangCopy="true"/>
<assetNode assetReferenceAttribute="fragmentPath"  resourceType="cq/experience-fragments/editor/components/experiencefragment"
           checkInChildNodes="false" createLangCopy="true"/>
```

- `(assetReferenceAttribute, resourceType)` → "this property, on this component, **is** a reference."
- `createLangCopy` → create a language copy of the target so the translated page links to it.
- `checkInChildNodes` ("Check child nodes for Asset reference" in the Rules UI) → the attribute may sit
  on an **untyped descendant** of the component node, e.g. teaser `actions/item0/link`; match it via the
  nearest ancestor that has its own `sling:resourceType`. It is *not* about recursing into the referenced
  target. Without honoring it, sub-node links stay pointing at the source language after import.

**Why this matters:** custom connectors typically hand-code reference finders that hardcode "component X
keeps its reference in attribute Y." `<assetNode>` externalizes that into config authors already own —
supporting a new referencing component is **one XML line in `/conf`, no bundle change/redeploy**. A
typical file ships ~14 entries (core/foundation/weretail/commerce/docs image, video, download,
content-fragment, experience-fragment).

**Its limits (keep custom code for these):** `<assetNode>` matches `(propertyName, resourceType)` on the
component node (or, with `checkInChildNodes`, its untyped sub-nodes). It does **not** cover references
embedded in rich-text HTML, transitive references inside the referenced target, or inherited (live-copy)
references. Treat it
as the declarative ~80%, with custom logic reserved for the edge cases.

---

## Practical notes

- **Attribute order varies** in generated files (often alphabetical) — parse order-independently.
- `<filter>` may be **empty** (`<filter/>`) — tolerate it.
- Multiple `<node path>` contexts and many `<assetNode>`/`<contentFilterNode>` entries are normal.
- Rule lookup is hot (per-node/per-property) — cache the parsed file and keep evaluation on the JCR API
  (`Node`/`Property`), not Sling. Caching it correctly on AEMaaCS needs three things, not one — see
  [translation-connector-cached-services.md](translation-connector-cached-services.md).

---

## Gotcha: a `ResourceChangeListener`-invalidated cache goes stale across author pods

- **Symptom:** an edited rule takes effect for some submissions and not others; the Translation Rules
  report shows a property as translatable while the generated XLIFF omits it. Heals itself at the next
  deploy. Never reproduces locally.
- **Root cause:** plain `ResourceChangeListener` "gets only local events" (Sling javadoc). AEMaaCS author
  is ≥2 pods on one repository, so the edit clears the cache only on the pod that handled the write. The
  report is a routed request, XLIFF serialization runs in a Sling job — different pods, different caches.
- **Fix:** also implement `ExternalResourceChangeListener` (a bare marker; keep registering the service
  under `ResourceChangeListener.class`), stamp the cached value with an invalidation counter sampled
  before the read, and add a TTL backstop.
- **Next time:** any cache invalidated only by a change listener is suspect on AEMaaCS — grep for
  `implements ResourceChangeListener` without the marker.

---

## Sources

Adobe's official documentation for the `<property>` attributes (`translate`, `inherit`,
`updateDestinationLanguage`). Both pages carry the same verbatim definition of `inherit`: *"By default,
every property is inherited, but if you want a property not to be inherited by the child, then you can
mark that property to be false so that it is applied only to that specific node."*

- [Identifying Content to Translate — AEM as a Cloud Service](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/sites/administering/reusing-content/translation/rules)
- [Identifying Content to Translate — AEM 6.5](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/sites/administering/introduction/tc-rules)