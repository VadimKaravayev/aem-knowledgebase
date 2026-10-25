# Repository Modernizer — legacy 6.x Maven project → AEMaaCS package layout

One of the code refactoring tools in the cloud transition toolkit ([[cloud-transition-tools-overview]]). It restructures a **legacy AEM 6.x Maven project** into the package split that AEM as a Cloud Service requires ([[aemaacs-content-package-structure]], [[filevault-package-types]]). It fixes **project shape**, not Java code.

## How to run it

- **Adobe I/O CLI**: `aio aem-migration:repository-modernizer` from `aio-cli-plugin-aem-cloud-service-migration` (same plugin as [[dispatcher-converter]] and the Index Converter) — see [[adobe-io-cli-overview]].
- **Standalone**: `@adobe/aem-cs-source-migration-repository-modernizer` npm package. Same code, GitHub repo under `adobe`.
- Reads a **`config.yaml`**, writes the restructured project to a **`target` folder plus a markdown summary report**, and leaves the source project untouched.

## `config.yaml` in one glance

| Key | Declares |
|---|---|
| `groupId` | Maven group for all generated modules |
| `parentPom` | path + artifactId / appTitle / version of the parent POM to rewrite |
| `all` | artifactId / appId / version of the generated container package |
| `projects[]` | one entry per existing project: `projectPath`, `existingContentPackageFolder[]`, `relativePathToExistingFilterXml`, `relativePathToExistingJcrRoot`, new `artifactId` / `appTitle` / `appId` / `version`, `osgiFoldersToRename` map |

## What it does

- **Splits each existing content package by filter path**: immutable content (`/apps`, `/oak:index`) → **`ui.apps`**; mutable content (`/content`, `/conf`, …) → **`ui.content`**.
- **Moves OSGi configuration folders** into a new **`ui.config`** module, renaming run-mode folders per `osgiFoldersToRename` so they land under `/apps/<app>/osgiconfig/config.<runmode>` ([[aemaacs-osgi-configuration]]).
- **Generates the `all` container package** that embeds the others ([[all-package-embed-structure]]) and a **`ui.apps.structure`** repository-structure package declaring the valid roots the FileVault validator checks ([[filevault-validation-compliance]]).
- **Rewrites the parent POM**: modules, `filevault-package-maven-plugin`, Cloud Manager target settings; removes plugins deprecated on Cloud.
- **Writes the summary report**: every move applied, plus everything it could not place — content under `/etc`, nested or non-standard packages, paths outside the known split.

## What it leaves to you

- The **`core` bundle and all Java code** — removed 6.x APIs, Guava, deprecated classes stay as they are ([[guava-deprecation-cleanup]]).
- **Uber-jar → `aem-sdk-api`** dependency swap ([[aemaacs-sdk]]).
- **OSGi config file format**: relocated, but `.config` / XML originals still need converting to `.cfg.json`.
- Dialog, template and component modernization (AEM Modernization Tools, a separate step).
- **Relocating content out of `/etc`** and `/var` — flagged, not fixed.
- Validation: run the build locally with the AEM Analyser plugin before the first Cloud Manager pipeline ([[aem-analyser-maven-plugin]]).

## Traps

- **Immutable vs mutable is decided purely by path.** A mixed package with `/apps` and `/content` under one filter splits correctly; code deliberately kept in `/etc` or `/var` is listed in the report and fails the Cloud Manager build until relocated.
- A project whose content packages lack a standard `jcr_root` + `filter.xml` pair needs the relative paths spelled out in `config.yaml`, or the tool silently skips that package.
- The generated `ui.apps.structure` only knows the roots it saw; new roots added later must be appended by hand or the "filter root's ancestor is not covered" validator error returns.
- Rerunning overwrites `target` — keep manual fixes in the real project, never in the tool output.
