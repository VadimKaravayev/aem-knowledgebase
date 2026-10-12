# Auditing a vendored `all` package before embedding it on AEMaaCS

Embedding a third-party or sample `all` package in your own container is a one-line `<embedded>`, and
the build stays green even when what's inside will break the environment. Before you embed, unpack it
**all the way down**, because problems hide two or three levels deep. If you find any, re-vendor its
subpackages one by one instead of embedding the vendor's `all`.

Related: embed rules in [[all-package-embed-structure]] and [[aemaacs-content-package-structure]],
the publish failure in [[publish-distribution-poison-package]], log configs in
[[osgi-config-runmode-resolution]].

**Sourcing:** a real case, Adobe's WKND SPA React 1.0.3 (GitHub release, built 2020-08) embedded in a
Cloud Manager aggregator on AEM 2026.8. Contents verified by unpacking. Effects verified in the build
log (analyser warnings) and the author `aemerror` log.

---

## Unpack recursively

```bash
mkdir -p x && unzip -q vendor.all.zip -d x/all
find x/all/jcr_root -name '*.zip' | while read -r z; do
  d="x/$(basename "$z" .zip)"; mkdir -p "$d"; unzip -q "$z" -d "$d"; done
# repeat for zips found inside those
for p in x/*/; do echo "== $p"; grep -hE 'key="(name|version|packageType|dependencies)"' "$p/META-INF/vault/properties.xml"
  grep -h '<filter' "$p/META-INF/vault/filter.xml"; done
find x -name '*.zip' -o -name '*.jar' | sed 's|.*/jcr_root||'
```

## What to look for

| Find | Why it breaks AEMaaCS | Seen in WKND SPA 1.0.3 |
|---|---|---|
| `core.wcm.components.all-*.zip` anywhere | AEMaaCS ships Core Components under `/libs` (2.30.x in 2026). A bundled copy puts an **older** bundle in `/apps/core/wcm/install` and **overrides `/apps/core/wcm/components` for every site** on the instance. | `spa.project.core.ui.apps-1.0.6` → `install/core.wcm.components.all-2.7.0.zip` |
| Subpackages under `/etc/packages/...` | 6.x-style nesting. AEMaaCS expects `<embeddeds>` under `/apps/<x>-packages/.../install`. | Inside core.wcm.components.all 2.7.0 (`adobe/cq60`) |
| `packageType=application` on an `all` that contains content packages | It's really a container. Embed it under `.../container/install`, not `application/install`. | `wknd-spa-react.all` |
| A run-mode-less `config/` log config at `debug` | On dev (no stage/prod config applies) it can flood `error.log`. | `LogManager.factory.config-wknd-spa-react` at debug + sample `SimpleResourceListener` = **half of a 325 MB/day author log** |
| Single-instance configs also set by another package (`JcrResourceResolverFactoryImpl`, `ReferrerFilter`, …) | The more specific run mode wins **the whole config**, so one site's setting is silently lost. Same run mode in both = "Configuration already defined" at Build Images. | SPA `config.publish` resource mapping vs WKND `config.publish.{dev,stage,prod}`: the SPA's is dead |
| Any `.content.xml` not starting with `<?xml` | Author accepts it, publish rejects it, and Deploy fails at Install mutable content after looping for hours. See [[publish-distribution-poison-package]]. | `ui.content` `.../spa-page/initial/.content.xml` |
| Build `created` date / `-SNAPSHOT` versions inside a "release" | Shows which platform it was built for. | `created 2020-08-24`, internal `1.0.3-SNAPSHOT` |

Note: the Cloud Manager analyser **only warns** about the bundled Core Components
(`[bundle-unversioned-packages] core.wcm.components.core:2.7.0 … (…|adobe/cq60:core.wcm.components.all:2.7.0)`).
The build and Build Images stay green, so read the warning's provenance chain.

## Re-vendoring recipe

1. Vendor each needed subpackage into a file-based Maven repo in the module (`<repository>
   file:${project.basedir}/local-repo`), with a minimal `.pom` per artifact.
2. To strip or fix a subpackage, unzip it, change it, and re-zip `META-INF` + `jcr_root`.
3. **Two versions, decided separately:**
   - **Maven version**: always add a suffix when you change anything (`1.0.6-without-core-components`,
     `1.0.3-xmlfix`), so nobody mistakes it for the vendor's artifact.
   - **Package version inside** (`properties.xml`): keep it if other subpackages declare a
     dependency on it (`ui.apps` depends on `spa.project.core.ui.apps:1.0.6`; `ui.content` depends on
     `ui.apps:1.0.3-SNAPSHOT`). Change it for a content package that was already installed or
     distributed, so the fixed one counts as new.
4. Embed the pieces directly: application packages under `.../application/install`, content under
   `.../content/install`. Install order comes from the subpackages' own declared dependencies, not
   from embed order.
5. Give each container **its own filter root** (`/apps/<container>-packages`). Two containers sharing
   a replace-mode root delete each other's embeds.
6. Repeat the audit on your own output: unzip the built container recursively and grep for
   `core.wcm.components|/apps/core/|etc/packages`. Expect zero matches.

## Checklist

- Unpack recursively. The problems live two or three levels down.
- No `core.wcm.components.all` in anything you ship to AEMaaCS.
- Check the vendor's run-mode-less log configs and single-instance configs against your other packages.
- `xmllint` every vendored `.content.xml`.
- Changed artifact = suffixed Maven version. Keep or change the internal version based on dependencies.
