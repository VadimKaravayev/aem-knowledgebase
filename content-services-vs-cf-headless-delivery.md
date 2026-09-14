# Content Services vs Content Fragment headless delivery paths

AEM has **two distinct headless/JSON delivery mechanisms**, and they consume
**different sources**. Conflating them is a classic architecture-exam trap and a
real design mistake.

## The two paths

| Path | What it serializes | Source it consumes | Typical consumer |
|------|--------------------|--------------------|------------------|
| **Content Services** (Sling Model / JSON Exporter, `.model.json`) | AEM **pages, components, and Experience Fragments** rendered to JSON | **Experience Fragment** (and pages/components) | SPA Editor apps, native mobile apps consuming component JSON |
| **Content Fragment delivery** (GraphQL API, Assets HTTP API `/api/assets`) | Structured **Content Fragment** data | **Content Fragment** directly | Pure headless clients querying structured content |

Key point: **Content Services is the Sling Model Exporter.** It turns a
*resource/component tree* into JSON. A Content Fragment is structured asset data,
**not** a component tree, so it is **not** the thing "Content Services" exposes.
Content Fragments go headless through **GraphQL or the Assets HTTP API** instead.

## Canonical "author once, reuse everywhere" content hub

The recommended hub-and-spoke when one editor team authors text once and reuses
it across web + headless + personalization:

```
                         Content Services (JSON Exporter)
                                    ^
                                    | (XF serialized to JSON)
   Brand Website  <---  Experience Fragment  <---  Content Fragment  --->  Adobe Target
        (HTML)            (web composition)        (single source           (content offer)
                                                     of truth)
                                    |
                                    +--> SPA / Mobile (via Content Services JSON)
```

- **Content Fragment = single source of truth.** Text entered & reviewed once.
- **CF -> Experience Fragment:** XF composes the CF text with layout/presentation.
- **XF -> Content Services -> SPA / Mobile:** the JSON Exporter serializes the XF.
  Headless apps consume the **Experience Fragment as JSON**, *not* the raw CF.
- **CF -> Adobe Target:** export the CF directly as a content offer.

## The trap (why "CF -> Content Services" is wrong)

It is tempting to wire **Content Fragment directly into Content Services** because
"headless needs structured CF data." But the "Content Services" box is the Sling
Model JSON Exporter, whose input is a component/XF tree. The correct edge feeding
Content Services is **Experience Fragment -> Content Services**. If you genuinely
want raw CF JSON, that is the **GraphQL / Assets API** path — a different box, not
"Content Services."

Direction also matters: the Content Fragment is the atomic, channel-agnostic
source and must **feed** the Experience Fragment (**CF -> XF**). Reversing it
(**XF -> CF**) makes the HTML-oriented XF the source of truth and breaks
"author once."

## Related
- See `oak-indexing-aemaacs.md` for how CF/XF content gets queried/indexed.