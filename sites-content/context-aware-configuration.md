# Context-Aware Configuration (CA Config)

Sling's mechanism (`org.apache.sling.caconfig`) for configuration that varies by **content context** — which site/brand/language subtree the current resource lives in — where OSGi config varies by **instance** (author/publish, dev/stage/prod run modes). The one-line decision rule: requirement says *"each brand/site/market needs..."* → CA Config; *"on publish instances / in prod..."* → OSGi. Run modes are a closed set on AEMaaCS and were never meant to encode tenancy ([rolling-deployment-two-version-overlap.md](../cloud-service/rolling-deployment-two-version-overlap.md)); CA Config is the supported multi-tenant axis.

## Mechanism: three pieces

1. **Content points at its config.** The context root's `jcr:content` carries `sling:configRef = /conf/brand-a`. Everything below inherits that context. (Site-creation wizards and the archetype usually set this — check before adding one by hand.)
2. **Config lives under `/conf`** in the `sling:configs` bucket: `/conf/brand-a/sling:configs/com.example.config.ContactConfig`. The node name is the config's name — by default the annotated interface's FQCN.
3. **Resolution walks up and falls back.** From any resource: nearest ancestor with `sling:configRef` → that `/conf` path; not found there → parent contexts → `/conf/global` → `/apps/conf` → `/libs/conf`. Define defaults once at `/conf/global` (or ship them in `/apps/conf`), override per brand/market only what differs.

Same `/conf/<site>` tree, different buckets: CA Config is `sling:configs`, editable templates are `settings/wcm`, cloud configs `settings/cloudconfigs`. Don't conflate them — and the Configuration Browser manages the *folders* and cloud configs, it is **not** a CA-config editor.

## Using it

**Define** (core bundle):

```java
@Configuration(label = "Contact Info", description = "Per-brand contact details")
public @interface ContactConfig {
    @Property(label = "Contact e-mail")
    String email() default "info@example.com";
    @Property(label = "Phone")
    String phone();
}
```

**Read** — always adapting the **content** resource (never the `/conf` node):

```java
ContactConfig cfg = resource.adaptTo(ConfigurationBuilder.class).as(ContactConfig.class);
```

Collections (repeatable items, e.g. social links): `@Configuration(collection = true)` on the annotation and `.asCollection(SocialLink.class)`; stored as child nodes under the config node. In a Sling Model, inject the request/resource and adapt; HTL gets it through such a model.

**Edit.** No rich OOTB authoring UI for arbitrary CA configs — the de-facto standard is the **wcm.io Context-Aware Configuration Editor** (separate open-source package; a page component dropped at e.g. `/content/brand-a/config` renders forms generated from the `@Property` annotations, with inheritance toggles). Without it, the config is maintained as repo content / deployed in `ui.content`.

## Inheritance is opt-in, and the default is the replace-not-merge trap again

- A config node present at brand level **masks** the global one entirely unless `sling:configPropertyInherit = true` is set on it — then missing properties merge from the fallback levels. Identical failure shape to run-mode OSGi folders ([osgi-config-runmode-resolution.md](../infrastructure-ops/osgi-config-runmode-resolution.md)): the override works, every property you *didn't* repeat silently reverts to default.
- Collections need their own switch (`sling:configCollectionInherit`, e.g. `distinct`) to combine items across levels instead of replacing the list.

## Gotchas

- **Works on author, defaults on publish**: `/conf` is content — it must be **published** like any page, and the publish-side session needs **read ACLs on `/conf`** (service users often have `/content` only). Both halves of that bug produce the same symptom: `as()` quietly returns annotation defaults, no error.
- **OSGi configs never go under `/conf`** — the OSGi installer reads `config` folders under `/apps` only; a `<PID>.cfg.json` under `/conf/...` is dead content. This is the location half of the duplicate-asset-detection exam trap ([duplicate-asset-detection.md](../assets-dam/duplicate-asset-detection.md)).
- **Deploying defaults without clobbering customer edits**: `/conf` shipped in `ui.content` is mutable content — a plain filter overwrites what authors changed on every deployment. Use merge-mode filters; Adobe's connector guidelines mandate "defaults must merge, never overwrite" ([aem-connector-implementation-guidelines.md](../java-osgi-build/aem-connector-implementation-guidelines.md), which also mandates CA Config as *the* way connector code reads its Cloud Services config).
- **Multi-tenant caching**: a service caching CA-config-derived values must key the cache by context path and invalidate globally across pods ([translation-connector-cached-services.md](../translation-i18n/translation-connector-cached-services.md)).

## Exam patterns

- Multi-brand / per-language / per-region overrides with shared defaults (contact info, approval groups) → **CA Config**; distractors are OSGi-per-runmode (can't vary by content path, run modes are coarse and closed), Configuration Browser (folder/cloud-config management, no inheritance model), one-workflow-per-combination (maintenance explosion).
- Any option placing an OSGi `.cfg.json` under `/conf` is wrong regardless of its other merits.

Sourced from Sling caconfig / Adobe docs and prior project experience, Oct 2026; code snippets not re-verified against a live instance for this doc.
