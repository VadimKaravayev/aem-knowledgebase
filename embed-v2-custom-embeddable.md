# Core Embed (v2): custom embeddables, and the headless/SPA export gotcha

## The component in one paragraph

`core/wcm/components/embed/v2/embed` gives authors three input types, each a different trust level:

1. **URL** — author pastes a widget URL; it's validated against registered `UrlProcessor`s (oEmbed processor covers Facebook, Flickr, Instagram, SoundCloud, Twitter, YouTube; a dedicated Pinterest processor ships too).
2. **Embeddable** — author picks from pre-configured trusted widgets (YouTube ships OOTB). Parameterized via dialog fields; may emit unsafe tags because the source is trusted.
3. **HTML** — free-form HTML, **sanitized to safe tags only** (scripts are stripped).

Template-level gating via the design dialog / content policy: `./urlDisabled`, `./embeddablesDisabled`, `./htmlDisabled`, and `./allowedEmbeddables` (multi-value list of embeddable resource types the author may pick). The author's choice is stored as `./type` plus `./url` / `./embeddableResourceType` / `./html`.

## Custom embeddable recipe

An embeddable is a hidden component that plugs its dialog fields into the *parent* Embed dialog:

1. Create a component with `sling:resourceSuperType = core/wcm/components/embed/v1/embed/embeddable` — **the v1 superType, even for the v2 embed; the README lies here.** `EmbeddablesDataSourceServlet` (populates the "Allowed Embeddables" select in the embed *policy design dialog*, serving both v1 AND v2 datasource RTs) runs an XPath query for the exact literal `sling:resourceSuperType='core/wcm/components/embed/v1/embed/embeddable'` (`Embed.RT_EMBEDDABLE_V1` — no v2 constant exists, no supertype-chain resolution, and v2's `embeddable` base doesn't chain to v1). A v2-superType embeddable is invisible to the policy editor; Adobe's own YouTube embeddable lives under v1 with the v1 superType. Verified on core components 2.30.4 via QueryBuilder. Hide it from the component browser — `componentGroup=".hidden"`.
2. Add an HTL rendering script named after the component (e.g. `customervoice.html`). The embed renders the embeddable by including the *embed resource itself* with your embeddable's resource type forced — so your HTL reads properties straight off the embed node.
3. Add a `cq:dialog` containing only your fields, and on it a `granite:data` node with:
   - `cmp-embed-dialog-edit-embeddableoptions="true"`
   - `cmp-embed-dialog-edit-showhidetargetvalue="<your embeddable resourceType>"`

   The parent Embed dialog merges these fields in and shows them only when your embeddable is selected in the type dropdown (same client-side showhide family as [[dialog-showhide-fields]]).
4. **Namespace every property name** (e.g. `msDynamics365SurveyId`, not `surveyId`) — all embeddables store their config on the *same* embed content node, so un-prefixed names collide across embeddables. The prefix does NOT have to equal the node name (avoid stutter like `msdynamics365customervoiceSurveyId`).
5. Optional `cq:design_dialog` with the same two `granite:data` properties if you need per-policy config; when your embeddable is in `allowedEmbeddables`, its design tab is merged into the embed policy dialog.

JCR shape (mirrors the OOTB YouTube one at `/libs/core/wcm/components/embed/v1/embed/embeddable/youtube`):

```
/apps/<app>/components/embed/embeddable/customervoice/
  .content.xml            (sling:resourceSuperType = core/.../embed/v1/embed/embeddable)
  _cq_dialog/.content.xml (granite:data props above; namespaced field names)
  _cq_design_dialog/      (optional)
  customervoice.html
```

For key/value config (e.g. survey context variables), use a composite multifield storing child nodes (`./myPrefixContext/itemN` with `key` + `value` properties) instead of a raw-JSON textarea — keys land as property *values*, so spaces/any characters are safe, and authors can't produce malformed JSON that silently drops.

## Policy is a hard prerequisite — no policy, no embeddable in the author dialog

All three embeddable datasources read **only** the embed component's content policy (`ContentPolicyManager.getPolicy(contentResource)`, falling back to `Designer` style):

| Datasource RT (`v1`/`v2`) | Feeds | Reads |
|---|---|---|
| `.../datasources/embeddables` | policy design dialog "Allowed Embeddables" select | XPath query for v1 superType (see recipe step 1) |
| `.../datasources/allowedembeddables` | edit dialog "Embeddable" dropdown | policy's `allowedEmbeddables` values, resolves each RT for its `jcr:title` |
| `.../datasources/embeddableoptions` | merges each embeddable's fields into the edit dialog | policy's `allowedEmbeddables`, resolves `<resourceType>/cq:dialog` per entry |

So a deployed, correctly-built embeddable shows **nothing** in the author dialog until the embed component has a content policy with your RT in `allowedEmbeddables`, mapped for the template (template editor, or code: policy node under `/conf/<app>/settings/wcm/policies/<embed RT>/policy_X` + a `wcm/core/components/policies/mapping` node under the template's `policies/jcr:content/<container-path>/<embed RT>`). Debug tip: render the merged dialog headlessly and grep for your fields — `curl -u admin:admin '/mnt/override/apps/<embed RT>/_cq_dialog.html/<embed content path>'`.

Renames touch three synced spots: the component node name, the `cmp-embed-dialog-edit-showhidetargetvalue` value in its dialog, and the policy's `allowedEmbeddables` entry — miss one and the fields stop appearing.

## The headless/SPA gotcha: embeddable properties are NOT exported to model.json

The Embed Sling Model (`EmbedImpl`, shared by v1/v2) exports exactly its interface getters: `type`, `url`, `result` (URL-processor output), `html`, `embeddableResourceType`, `:type`. **The embeddable's namespaced properties never appear in model.json** — they exist only on the JCR node, consumed server-side by the embeddable's HTL, which a SPA never runs.

Verified empirically (AEM 6.5 LTS, core components deployed in `/libs`):

```
# create a scratch node with the YouTube embeddable configured
curl -u admin:admin -F "sling:resourceType=core/wcm/components/embed/v2/embed" \
  -F "type=embeddable" \
  -F "embeddableResourceType=core/wcm/components/embed/v2/embed/embeddable/youtube" \
  -F "youtubeVideoId=dQw4w9WgXcQ" http://localhost:4502/content/embed-model-test

curl -u admin:admin http://localhost:4502/content/embed-model-test.model.json
# → {"id":"embed-12e46f5f1c","type":"EMBEDDABLE",
#    "embeddableResourceType":"core/.../embeddable/youtube",
#    ":type":"core/wcm/components/embed/v2/embed"}     ← youtubeVideoId is absent
```

Consequences for SPA/headless projects:

- A custom embeddable is **authoring-side only** OOTB. To use one headlessly you must add a custom Sling Model for your embed proxy (delegating to / extending the core `Embed` model) that additionally exports the embeddable's config. **Prefer a typed model over a generic property map** — a map filtered by prefix/reserved-names leaks stale node properties, has no contract with the SPA, and dialog field renames silently change the JSON API. Working pattern: a small `@Model(adaptables = Resource.class)` per embeddable with `@ValueMapValue(name = "<namespaced prop>")` getters (multifield children via `@ChildResource` → `LinkedHashMap` preserves authoring order), plus one getter on the delegating embed model — `@JsonProperty("myEmbeddable") @JsonInclude(NON_NULL)` returning `resource.adaptTo(MyEmbeddable.class)` only when `delegate.getType() == EMBEDDABLE` **and** `delegate.getEmbeddableResourceType()` equals your RT (else null, so other embeddables/types don't export it).
- The `type` enum serializes as uppercase (`"URL"` / `"EMBEDDABLE"` / `"HTML"`).
- The **HTML type is doubly useless for scripts in a SPA**: AEM sanitizes the stored HTML to safe tags (scripts stripped server-side), and React frontends typically render it via `dangerouslySetInnerHTML`, which never executes `<script>` tags anyway. Third-party JS widgets belong in an embeddable (or a dedicated component), with the SPA loading the vendor script itself.

## Security notes (from Adobe's docs)

Register processors/embeddables only for trusted sources, HTTPS only, and avoid the `unsafe` HTL display context unless the payload is verified trustworthy.

Refs: component README `adobe/aem-core-wcm-components` → `content/.../embed/v2/embed`; model source `internal/models/v1/EmbedImpl.java`. Confirmed on AEM 6.5 LTS local author, July 2026.