# Cloud Manager (AMS) build environment

What the AMS / AEM 6.x Cloud Manager **build container** looks like and how a
pipeline build actually runs. The three variable *mechanisms* (CM-set build
vars, custom pipeline vars, runtime env vars) are in
[[cloud-manager-variables]]; pipeline configuration in
[[cloud-manager-ams-production-pipelines]]. AEMaaCS has its own
build-environment page with different Java versions — **JDK 8/11 here is the
AMS tell** (cloud builds are JDK 11+).

---

## The container

- Linux, **derived from Ubuntu 22.04**; a **fresh container per build**,
  nothing persists between executions.
- **Maven 3.9.4**, configured system-wide with a `settings.xml` carrying the
  **`adobe-public`** repository profile. Maven ≥ 3.8.1 blocks plain-HTTP
  repositories — custom repo URLs must be **HTTPS** or the resolve step
  fails.
- Pre-installed system packages: `bzip2`, `unzip`, `libpng`,
  `imagemagick`, `graphicsmagick`. Anything else can be apt-installed at
  build time via `exec-maven-plugin` — but such packages exist **only in the
  build container, never in the AEM runtime** (the classic wrong answer to
  "how do I get ImageMagick onto AMS publish").

## Java selection

- Installed: **Oracle JDK 8u401** (`/usr/lib/jvm/jdk1.8.0_401`, the
  **default**) and **Oracle JDK 11.0.22** (`/usr/lib/jvm/jdk-11.0.22`).
- Select with a **`.cloudmanager/java-version`** file in the Git branch,
  content exactly `8` or `11` — sets the JDK and `JAVA_HOME`.
- **Maven Toolchains support was removed in Cloud Manager 2025.06.0**: a
  pipeline still using `maven-toolchains-plugin` fails with
  `Cannot find matching toolchain definitions`. That error on a
  previously-green build = migrate to the `java-version` file, not a
  toolchains.xml fix.

## How the build runs

Three sequential Maven invocations, all `--batch-mode`:

1. `maven-dependency-plugin:3.1.2:resolve-plugins`
2. `maven-clean-plugin:3.1.0:clean -Dmaven.clean.failOnError=false`
3. **`org.jacoco:jacoco-maven-plugin:prepare-agent package`**

Consequences: the unit-test coverage agent is injected by Cloud Manager
itself, so a project pinning **JaCoCo < 0.7.5.201505241946** breaks the
build; and the target is plain `package` — no `install`/`deploy`, so
inter-module expectations relying on the local repo of a previous build
don't hold (fresh container).

## Variables during the build

Always present: `CM_BUILD` (= `true`), **`BRANCH`**, `CM_PIPELINE_ID`,
`CM_PIPELINE_NAME`, `CM_PROGRAM_ID`, `CM_PROGRAM_NAME`;
`ARTIFACTS_VERSION` only on stage/production pipelines
(activation idioms in [[cloud-manager-variables]]).

Custom pipeline variables (`aio cloudmanager:set-pipeline-variables
PIPELINEID --variable NAME value [--secret …]`) have hard limits:
**200 variables per pipeline**, name **< 100 chars**
(alphanumeric + underscore), string value **< 2048 chars**, secret value
**< 500 chars**.

## Exam checklist

- Fresh Ubuntu-derived container per build; nothing persists.
- JDK **8 default**, 11 via `.cloudmanager/java-version`; **toolchains
  removed 2025.06.0** (`Cannot find matching toolchain definitions`).
- Maven 3.9.4, HTTPS-only repos, `adobe-public` profile preconfigured.
- Build = resolve-plugins → clean → **jacoco prepare-agent package**;
  JaCoCo ≥ 0.7.5.201505241946.
- apt packages at build time never reach the AEM runtime.
- Pipeline vars: ≤ 200, name < 100, value < 2048, secret < 500.

## References
- [Build environment (Adobe docs, Cloud Manager for AMS)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-manager/content/getting-started/project-creation/build-environment)

Docs-based (page read Oct 2026), not verified against a live AMS build. The
page summary also surfaced AEMaaCS-flavored items (author/preview/publish
secret-vs-variable contexts, Node 18 front-end pipelines) that belong to the
cloud build-environment doc set; they are deliberately left out here rather
than attributed to AMS.
