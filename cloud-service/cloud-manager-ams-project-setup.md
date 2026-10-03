# Cloud Manager (AMS) project setup

How a Maven project must be shaped for Cloud Manager to build and deploy it
(AMS / AEM 6.x doc set; the mechanisms largely mirror AEMaaCS). The build
*container* is [[cloud-manager-ams-build-environment]]; the `env.CM_BUILD`
profile-activation idioms are in [[cloud-manager-variables]]; pipeline
configuration in [[cloud-manager-ams-production-pipelines]].

---

## Structure and artifact discovery

- **`pom.xml` at the Git repository root**, sub-modules and extra Maven
  artifact repositories allowed.
- Cloud Manager finds deployable **content packages by scanning for `.zip`
  files under `target/` directories** of the built modules; dispatcher
  artifacts are picked up from `target/conf` and `target/conf.d`.
- With several modules producing packages, **deployment order is not
  guaranteed** — if order matters, declare **content-package dependencies**
  between them (the package `dependencies` config, not Maven module order).

## Keeping a package out of the deployment

Set **`<cloudManagerTarget>none</cloudManagerTarget>`** in the package
plugin's properties — honored by both `filevault-package-maven-plugin` and
the legacy `content-package-maven-plugin`. This is the switch for packages
that must be built (e.g. test content, local-only tooling) but never
deployed by the pipeline.

## Password-protected Maven repositories

Credentials go in **pipeline variables** (secret ones for passwords,
[[cloud-manager-ams-build-environment]] for limits) referenced from a
**`.cloudmanager/maven/settings.xml`** in the repo (standard Maven settings
schema). Server `id` rules: the prefixes **`adobe` and `cloud-manager` are
reserved**, and Cloud Manager's own mirror intercepts the default
**`central`** id — so give the private repo a distinct id or its requests
are mirrored away from your host.

## Build artifact reuse

Within one **program**, a pipeline run on the **same Git commit hash**
reuses the previously built artifact instead of rebuilding — including
**across branches** (same commit reachable from two branches = one build),
but never across programs. Opt out per pipeline with the variable
**`CM_DISABLE_BUILD_REUSE=true`** — the knob when a build is
environment-sensitive (profiles keyed on anything beyond the commit), which
is itself a smell: reused artifacts assume the commit fully determines the
output.

## Exam checklist

- Root `pom.xml`; packages discovered as `target/**.zip`; dispatcher config
  from `target/conf`+`conf.d`.
- Multi-package deploy order = content-package dependencies, not module
  order.
- `cloudManagerTarget=none` skips deployment of a built package.
- Private repo creds: pipeline variables + `.cloudmanager/maven/settings.xml`;
  `adobe`/`cloud-manager` id prefixes reserved, `central` is mirrored.
- Same commit in one program → artifact reuse (cross-branch, not
  cross-program); disable with `CM_DISABLE_BUILD_REUSE=true`.

## References
- [Project setup (Adobe docs, Cloud Manager for AMS)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-manager/content/getting-started/project-creation/project-setup)

Docs-based (page read Oct 2026), not verified against a live AMS program.
