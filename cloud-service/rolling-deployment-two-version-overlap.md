# AEMaaCS rolling deployments: designing for the two-version overlap

An AEMaaCS deploy is a rolling one. New nodes start and pass health checks while the **old version is
still serving traffic**, then old nodes retire. There is no moment when exactly one version of your
code is live, so any change that assumes "my new code and my new repository state arrive together"
is wrong. Mutable content is applied only after the application version has switched over.

This is the constraint that shapes how a connector ships service users, ACLs and indexes.

**Sourcing:** the Adobe *Deploying to AEM as a Cloud Service / Overview* page (Sept 2026, re-read
2026-10-25 against the 2026-10-09 revision), interpreted for a vendor package. Not reproduced against a
live pipeline. This doc is the knowledgebase's home for that page; the sections below the three rules
carry its remaining reference facts.

---

## /apps and /libs are rebuilt from scratch, not diffed against the old image

The immutable half of the repository is not "updated" in the sense of applying a
delta to what was there before. Each pipeline run builds a **fresh container image**
from the repo at that commit, and the new pods start from that image. There is no
step that merges the new build's `/apps` content with the previous image's `/apps`
content — whatever isn't part of the current build's `ui.apps`/`ui.apps.structure`
packages simply isn't in the new image.

Consequence: anything ever placed under `/apps` outside of what the pipeline builds
(a manual hotfix, leftover code from a package installed by hand pre-migration) does
not survive the next deploy — not because it was explicitly deleted, but because the
new pods never had it to begin with. This is also why a rollback restores code but
not content (rule 2 below): the rollback just points traffic back at a previously
built image, and images are also swapped wholesale, never patched.

**Sourcing:** reasoned from AEMaaCS's documented immutable/mutable split and rolling
deploy model; not confirmed against Adobe's build-pipeline internals directly.

## What lands, and when

Mutable content does not install in one step. Three phases, in this order:

| Phase | What installs |
|---|---|
| **Pre-startup** | Oak index definitions |
| **During startup** | Service users, ACLs, node types (this is where repoinit runs) |
| **Post-switchover** | Everything else, via Jackrabbit Vault |

Repoinit therefore runs on a repository where **no package content exists yet**, which is why ACLs
must target paths that already exist. See
[repoinit-acls-on-apps-and-libs.md](repoinit-acls-on-apps-and-libs.md).

## The three rules that follow

### 1. Spread service user and ACL changes over two releases

While the deploy is in flight, **old code is running against the new permissions**. Narrowing a
grant, renaming a service user, or moving a subservice mapping in the same release that updates the
code that depends on it means the old nodes lose access mid-deploy and throw `LoginException` or
`AccessDeniedException` until they retire.

The two-release shape:

- **Release N**: add the new grant / new service user / new mapping. Both old and new code work.
- **Release N+1**: switch the code over, and only then remove what is now unused.

### 2. A rollback does not roll back content

Rolling back restores the **code**. Mutable content structures created by the failed release stay in
the repository. So the old code has to tolerate structures it did not create, or you fix forward with
a new release. There is no automatic cleanup and no way to undo a mutable change through the
pipeline.

Practical reading for a connector: never write a mutable structure the previous version would choke
on (a new property with a value the old enum cannot parse, a node type the old code does not expect
under a path it scans). Make the old version's readers tolerant *before* the release that writes.

### 3. Indexes set the length of the deploy

A new or modified index definition triggers **reindexing before traffic shifts**, so deploy duration
tracks index size, not code size. A large index change can turn a routine deploy into a long one.
During the overlap, old nodes keep using the original index while new nodes use the modified one, so
both definitions must be serviceable at once. See
[oak-indexing-aemaacs.md](oak-indexing-aemaacs.md) for the `-custom-N` naming convention that makes
this work, and note that ACS Commons Ensure Oak Index is **not** usable on AEMaaCS.

## Reference facts worth not rediscovering

- **Run modes are a closed set.** `author` and `publish` crossed with `rde`, `dev`, `stage`, `prod`,
  combined as `<service>.<environment>` (`author.dev`, `publish.prod`). No custom run modes, and **no
  way to scope a package to one environment**; environment-specific content means a manual Package
  Manager install. Bundles only get the service dimension (`install.author`/`install.publish`).
- **`getRunModes()` no longer reports the environment type at runtime.** Code branching on
  `dev`/`stage`/`prod` from the SlingSettings API silently misbehaves on AEMaaCS — the environment
  dimension exists only for OSGi config resolution. Branch via an OSGi config property set per
  run-mode folder instead.
- **`cp2fm` packages are frozen.** Anything Cloud Manager deployed appears in Package Manager with a
  `cp2fm` suffix and cannot be rebuilt, reinstalled or downloaded.
- **Manual one-off installs are mutable-only and time out after 10 minutes** — but the timeout's
  **"undefined" error is the Cloud Service request limit, not a failure**: the installation keeps
  running in the background. **Do not retry**; a second concurrent import of the same package is
  what actually creates conflicts.
- **No FileVault install hooks in immutable packages.** Mutable packages installed post-switchover
  *do* support FileVault install hooks.
- **A mixed immutable/mutable package installs only its mutable half**, silently at runtime — but
  the build does warn: `Generated content-package <PACKAGE_ID> … is of MIXED type` in the Cloud
  Manager build log. Keep the split clean in the `all` container.
- **OSGi config belongs in source control**, not the web console, and **maintenance task
  configuration must too**, because Tools → Operations is unavailable on AEMaaCS.
- Cloud Manager converts content packages into **Sling Feature Model** artifacts; every embedded
  third-party package must itself satisfy the cloud coding guidelines or the deploy fails. Two
  sanctioned ways to pull a third-party package in: reference it from a **remote Maven repository**
  (public or password-protected, creds per the pipeline-variable mechanism) or commit a
  **filesystem-based Maven repository** into the project and embed via
  `filevault-package-maven-plugin`.
- The quickstart's replication config stays reachable at **`/etc/replication`** on the local SDK
  even where the UI entry point is gone — publication APIs remain backward compatible with the
  AEM Replication Java APIs.

## The rest of the overview page, condensed

- **Two kinds of release, decoupled.** Custom code releases and **AEM version updates are separate
  deployment events**; Adobe's automatic updates are meant to be backward compatible with deployed
  custom code. Build against the **latest SDK** for internal releases and test against the AEM
  version currently on production ([[aemaacs-sdk]], [[local-development-environment]]); local and
  cloud behave differently, so dev and stage testing is not optional ([[environment-tiers-dev-stage-prod]]).
- **The only mutable state application code may depend on at startup**: service users, their ACLs,
  node types and index definitions — i.e. exactly what installs before or during startup. Anything
  else (folders, templates, `/conf`) arrives post-switchover and code must tolerate its absence.
- **Legacy `/etc` content must be relocated** (AEM 6.5 repository restructuring) before a codebase is
  cloud-deployable; the tooling is [[repository-modernizer]].
- **Two sanctioned OSGi config workflows**: edit in the local Configuration Manager and export into the
  project, or write the config by hand and use the console only to look up property names
  ([[aemaacs-osgi-configuration]]). Never manage config at runtime on cloud.
- **Mutable content's two vehicles**: Vault content packages (folder hierarchies, editable templates,
  `/conf`, scripts) and **repoinit**. Repoinit is for service users (create/delete/disable), groups,
  users, ACLs (**the path must already exist — precede with `create path`**), root folder structures
  and CNDs. Adobe's reasons to prefer it: created **at startup so code can rely on it**, explicit and
  additive (removals only for users/service users/groups), **fast and atomic** (a package install
  snapshots the whole tree even for one node), validated at runtime locally, **skipped when the state
  already matches**, and executed by Cloud Manager **independently of package installation**. Config
  = factory PID `org.apache.sling.jcr.repoinit.RepositoryInitializer~<descriptive-name>` with a
  `script` property; create parents before children; keep ACLs high and scope with `rep:glob`
  ([[repoinit-acls-on-apps-and-libs]]).
- **Author/publish-only packages**: embed under `install.author` / `install.publish` in `/apps`
  ([[all-package-embed-structure]]).
- **Package Manager on cloud** is a runtime tool for mutable one-offs (e.g. copying prod content to
  stage for a repro); it can never write the immutable repository ([[package-manager]]).
- **Third-party packages**: prefer a remote Maven repo; the committed file-system alternative is
  declared in the POM as

  ```xml
  <repository>
      <id>project.local</id>
      <name>project</name>
      <url>file:${maven.multiModuleProjectDirectory}/repository</url>
  </repository>
  ```

  and embedded via `filevault-package-maven-plugin` `embeddeds` to e.g.
  `/apps/vendor-packages/container/install`.
- **Rolling deploy, step by step**: old nodes active → release candidate built → indexes
  (re)processed, old nodes on old indexes → new nodes start → health checks while old nodes serve →
  ready new nodes take traffic, old nodes shut down → only new nodes remain → **mutable content
  deployed last**. Cloud Manager notifies when the new version is ready for traffic; index status is
  visible per build. **Canary deployments** are the alternative: validate the build on production
  infrastructure before routing live traffic (Adobe page *Use Canary Deployments to Validate Code*).
- **Replication**: the publication mechanism is backward compatible with the Replication Java APIs
  ([[content-distribution-journal]]); locally, classic author→publish replication is configured at
  `http://localhost:4502/etc/replication` if the UI entry point is gone.
- **Supported OSGi config folders — the full closed list**: `config`, `config.author`,
  `config.author.dev|rde|stage|prod`, `config.publish`, `config.publish.dev|rde|stage|prod`,
  `config.dev`, `config.rde`, `config.stage`, `config.prod`. Most matching run modes wins. Locally:
  `java -jar aem-sdk-quickstart-….jar -r publish,dev`.
- **RDE** receives code by CLI sync and skips the pipeline's quality gates; the regular dev
  environment is where the gates run before stage ([[cloud-manager-environments]]).
- **Adobe's own summary of restrictions**: no direct deploy to a running instance; no install hooks in
  immutable packages; mutable packages cannot target one environment; mutable changes cannot be
  auto-rolled back; non-cloud-compatible third-party packages fail the deploy; ACS Commons Ensure Oak
  Index unsupported; Package Manager "undefined" after ten minutes is a timeout, do not retry;
  `cp2fm` packages cannot be reinstalled, rebuilt or downloaded; run modes cannot target content.

## References

- [Deploying to AEM as a Cloud Service, Overview (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/deploying/overview)
- [repoinit-acls-on-apps-and-libs.md](repoinit-acls-on-apps-and-libs.md)
- [oak-indexing-aemaacs.md](oak-indexing-aemaacs.md)
- [cloud-manager-git-repositories.md](cloud-manager-git-repositories.md)
