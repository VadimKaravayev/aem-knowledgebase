# Content Fragment Enumeration fields and translation

Enumeration (dropdown) fields in CF models cannot be marked Translatable — by design, not
oversight. Only text-type fields get the Translatable checkbox; enum option label/value pairs
live in the *model* and are, per an Adobe employee, "only manually editable as an OOTB
functionality, not dynamic in nature." Example: WKND's `adventure` CF stores
`Day Trip` / `Overnight Trip` / `Training Session` — authors want them translated, AEM won't.

## Why the stored value must NOT be translated

The stored enum value is a **contract, not content**:

- It must match the model's option list — a translated value fails validation on reopen.
- GraphQL/headless clients filter and switch on it.
- Search facets, personalization, and any `value == "Day Trip"` comparison break.

A documented failure shape: the translation framework picks up the stored key, sends it out,
and writes the translated string back onto the language copy — dropdowns then fail to resolve.
A connector that force-translated enum properties would corrupt fragments.

## The four handling patterns (in order of recommendation)

1. **i18n dictionary at render time** — the mainstream, accepted answer. Stored value stays
   stable and doubles as the i18n key. Sites/HTL: `${properties.tripType @ i18n}` against
   per-language dictionaries. Headless: the frontend owns a per-locale key→label map (or
   fetches AEM's dictionary JSON). Bonus: the dictionaries themselves are translatable
   content — a translation vendor/connector can carry them as first-class material.
2. **Tags instead of Enumeration** — `cq:Tag` titles support per-language localized titles
   natively; zero custom code at display time. Best when the value set is really a taxonomy.
3. **Custom datasource servlet on the model dropdown** — localized *labels* in the authoring
   UI while storing the stable value. Authoring-side only; complements pattern 1, never
   replaces it.
4. **Custom translation-workflow key↔label mapping** around export/import — exists in the
   wild, recommended by no one; fragile and reintroduces the validation problem.

## Connector implication

Not serializing enum fields for translation is *correct* behavior. The customer-facing answer
is patterns 1/2; the product opportunity is translating i18n dictionaries, not bending enum
serialization.

## How the core CF component renders enums (verified in source, July 2026)

`core/wcm/components/contentfragment/v1/contentfragment` does NO enum handling — the stored
key renders literally, even on localized pages:

- `DAMContentFragmentImpl.DAMContentElementImpl.getValue()` returns the raw `FragmentData`
  value; nothing ever consults the model's option labels or an i18n dictionary.
- `element.html` branches on `dataType` only for `calendar` (formatted) and `boolean`; an
  enumeration's dataType is plain `string`, so it hits the raw-print fallback
  `${(element.value) @join='<br/>'}`. The `metaType` (which would say "enumeration") is used
  solely for single/multi-line text detection.
- The only i18n in the component localizes the *calendar format pattern*.
- The Sling Model Exporter (`.model.json`) path exports the same raw value — headless
  consumers get the untranslated key too.

**Cheapest render-time fix with core components:** projects consume these via proxy
components, and `element.html` is a small standalone HTL template — override just it in the
proxy and change the fallback branch to `${element.value @ i18n}` (per-language dictionaries
keyed by the option values). One line; stored value stays untouched (pattern 1 above).

## Sources

- [Identifying Content to Translate (Adobe docs — Translatable field / Enable Content Model Fields for Translation)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/sites/administering/reusing-content/translation/rules)
- [Translation Handling for Custom Dropdown Metadata (community — accepted answers: i18n dictionaries, datasource servlet, tags)](https://experienceleaguecommunities.adobe.com/adobe-experience-manager-sites-8/translation-handling-for-custom-dropdown-metadata-in-aem-assets-32967)
- [Content fragment model on translation (Adobe employee: enum options not translatable OOTB)](https://experienceleaguecommunities.adobe.com/t5/adobe-experience-manager/content-fragment-model-on-translation/td-p/565902)
- [Enumeration field validation error thread (stored value must match model options)](https://experienceleaguecommunities.adobe.com/t5/adobe-experience-manager/enumeration-data-type-content-fragment-throws-validation-error/td-p/449705)
- [Localized content with AEM Headless (locale folders + `_locale` filter; no enum translation mechanism)](https://experienceleague.adobe.com/en/docs/experience-manager-learn/getting-started-with-aem-headless/how-to/localized-content)

Researched July 2026.
