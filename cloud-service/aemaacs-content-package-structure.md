# AEMaaCS project content-package structure

The structural contract an AEMaaCS Maven project must follow — package
types, the `all` container's embed rules, the repository structure package.
Mixed-package install behavior in [[rolling-deployment-two-version-overlap]];
index packaging gotchas in [[oak-indexing-aemaacs]]; OSGi/repoinit content in
[[aemaacs-osgi-configuration]] and [[repoinit-acls-on-apps-and-libs]]; how
Cloud Manager discovers the artifacts in [[cloud-manager-ams-project-setup]].

---

## The four packages

| Package | `packageType` | Deploys to | Holds |
|---|---|---|---|
| **ui.apps** | `application` | `/apps` only (immutable) | components, HTL, clientlibs, `/libs` overlays, `/apps/settings` CA-configs, ACLs, precompiled scripts |
| **ui.content** | `content` | mutable: `/conf`, `/content`, `/etc`, tags | CA configs, baseline content structures, taxonomies |
| **ui.config** | `application` | `/apps/<app>/osgiconfig` | `config` + `config.<service>.<env>` folders, repoinit factory configs |
| **all** | *(container — none)* | nothing of its own | embeds everything; the **only** package Cloud Manager deploys |

Every non-container package sets
`<cloudManagerTarget>none</cloudManagerTarget>` so Cloud Manager deploys
only the `all` artifact.

Mutable vs immutable: `/apps` + `/libs` immutable; `/content /conf /var
/etc /oak:index /system /tmp` mutable — **but `/oak:index` ships in the
code package anyway**, because reindexing must finish before the image
cutover ([[rolling-deployment-two-version-overlap]]).

## The `all` container rules

- **`<embeddeds>`, never `<subPackages>`**, in the filevault plugin.
- Embed target path convention:
  `/apps/<app>-packages/(application|content|container)/install(.author|.publish)?`
  — four levels: `/apps` → `<app>-packages` → package-category folder →
  install scope.
- **The `-packages` suffix is load-bearing**: it keeps the embed target
  outside every real code package's `/apps/<app>/...` filter, so deploying
  the container can't overwrite a subpackage's own content. An embed
  targeted at `/apps/my-app/install` is the classic self-clobbering
  mistake.
- The container's own `filter.xml` must include the embed root
  (`<filter root="/apps/my-app-packages"/>`).
- `type: zip` for packages, `type: jar` for OSGi bundles.
- Housekeeping: bind `maven-clean-plugin` to `initialize` so stale embeds
  don't ride along in `target/`.

## Repository structure package

Required for **every `application` (code) package**, declared via
`<repositoryStructurePackages>` in the filevault plugin: it enforces that
one code package doesn't install over another's intermediate paths
(structural-dependency correctness). **Content packages don't need it.**
(And per [[oak-indexing-aemaacs]]: never put `/oak:index` in it.)

## Dependencies and order

- Rule: **mutable depends on immutable** — `ui.content` declares a package
  dependency on `ui.apps`; exception: a code package holding only OSGi
  bundles gets no package dependencies pointed at it.
- Multi-site: parallel `site-a.*`/`site-b.*` sets plus a shared
  `common.ui.apps`, all embedded in one `all`; cross-package install order
  = package dependencies, not embed order
  ([[cloud-manager-ams-project-setup]]).

## Repoinit packaging detail

Repoinit ships as an OSGi **factory config in `ui.config`**:
`…/osgiconfig/config.<runmode>/org.apache.sling.jcr.repoinit.RepositoryInitializer-<suffix>.config`
with a multi-line `scripts` property — **the `references` property does not
work on AEMaaCS**, inline `scripts` only. (Page shows `.config` format for
these; the rest of ui.config is `.cfg.json`,
[[aemaacs-osgi-configuration]].)

## Third-party packages

Embed the vendor's **`all` container** (not its pieces) into your `all`,
target under `/apps/vendor-packages/container/install`; the artifact must
come from a reachable Maven repository declared in the reactor pom
(auth via [[cloud-manager-ams-project-setup]]'s settings.xml mechanism).

## Misc

- `AccessControlHandling`: typically **`merge`** for both code and content
  packages (ACEs add without clobbering existing ones).
- `ui.apps` declaring the repository-structure dependency + `ui.content`
  depending on `ui.apps` is what makes a fresh-environment install land in
  the right order.

## Exam checklist

- Four packages; only `all` deploys (`cloudManagerTarget=none` on the
  rest); container embeds via `<embeddeds>`, never `<subPackages>`.
- Embed path = `/apps/<app>-packages/<category>/install[.service]`; the
  `-packages` suffix prevents self-overwrite.
- Repository structure package: code packages only.
- `/oak:index` is mutable at runtime but **deploys as code** (pre-cutover
  reindex).
- Repoinit: inline `scripts` in a factory config; `references` doesn't
  work.
- Third parties: embed their `all`, from a declared Maven repo.

## References
- [AEM Project Content Package Structure (Adobe docs, AEMaaCS)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/developing/aem-project-content-package-structure)

Docs-based (page read Oct 2026), not validated against a fresh archetype
build.
