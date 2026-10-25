# Local development environment for AEMaaCS — the SDK tier below RDE

The local environment is the **SDK Quickstart jar on your machine**, plus optional Dispatcher tooling: the fastest and cheapest tier before an RDE or a pipeline ([[environment-tiers-dev-stage-prod]]). Orientation page; artifact sources, version discipline and the CryptoSupport recipe are in [[aemaacs-sdk]].

## Components

| Piece | What it is | Notes |
|---|---|---|
| **Quickstart jar** | the AEM runtime, author *or* publish | Software Distribution Portal. Rename to `aem-author-p4502.jar` / `aem-publish-p4503.jar` — **the filename sets tier, run mode and port** — then `java -jar`. Adobe's prerequisite is **JDK 21** (JDK 11 only for old SDKs; see [[aem-sdk-java-ceiling-dual-65-cloud]]) |
| **`aem-sdk-api`** | compile-time API jar, successor of the 6.5 uber-jar, `provided` scope | **version must match production AEM** (About dialog / Cloud Manager env card) |
| **Dispatcher Tools** | validator for the dispatcher module + Docker image running Apache + Dispatcher in front of local publish, hot-reloads config | same SDK zip; what Cloud Manager's Build step validates against ([[dispatcher-fundamentals]], [[dispatcher-converter]]) |
| **Build** | `mvn clean install -PautoInstallSinglePackage` deploys the `all` package via Package Manager | runs the **same AEM Analyser** as Cloud Manager, so a clean local build catches most Build Images failures ([[aem-analyser-maven-plugin]], [[cloud-manager-pipeline-steps]]) |

## Runtime setup specifics (Adobe tutorial *Set up local AEM SDK*, read Oct 2026)

- **Download**: Software Distribution → *AEM as a Cloud Service* tab → sort by published date → latest *AEM SDK* → unzip `aem-sdk-XXX.zip` → `aem-sdk-quickstart-XXX.jar`. Needs an Adobe ID in an org provisioned for AEMaaCS.
- **Layout**: `~/aem-sdk/author/aem-author-p4502.jar` and `~/aem-sdk/publish/aem-publish-p4503.jar`, started with `java -jar …` from inside each folder; admin password `admin` for local. First start takes minutes.
- **Filename grammar `aem-<tier>_<environment>-p<port>.jar`**: `aem-author-p4502.jar` = author, `dev`, 4502; `aem-author_stage-p4502.jar` / `aem-author_prod-p4502.jar` switch the environment run mode; same for `aem-publish[_env]-p4503.jar`. **Once a tier has started it cannot be switched** — delete `crx-quickstart` and start again. Environment or port changes need a stop/restart. Use `dev` day to day; 4502/4503 are convention, other ports may need SDK config changes.
- **`-r prerelease` on the first start** enables upcoming-feature mode on either tier.
- **Deploy to both tiers**: `mvn clean install -PautoInstallSinglePackage -PautoInstallSinglePackagePublish` (WKND or your project).
- **Local "publishing"**: the cloud uses Sling Content Distribution + the Adobe Pipeline; **the legacy replication agents exist only in the local Quickstart jar** so you can simulate it. Author → `/etc/replication/agents.author.html` → *Default Agent (publish)* → Edit → Settings: *Enabled*, Agent User Id empty → Transport: URI `http://localhost:4503/bin/receive?sling:authRequestLogin=1`, user/password `admin`. Publish login for logs/configs: `http://localhost:4503/libs/granite/core/content/login.html`.
- **Stop** with Ctrl-C and wait for shutdown to finish.
- **Update cadence**: at least monthly, on or just after the **last Thursday of the month** (the AEMaaCS feature-release day). **Updating replaces the whole local environment — all local code, config and content is lost.** Commit code to Git first; for content, keep a sample-content package in Git, or `oak-upgrade` with `includepaths`, or export packages via Package Manager — and Adobe calls depending on any of these an anti-pattern: non-disposable code lives in the IDE and reaches AEM only by deployment.
- **Troubleshooting**: double-clicking the jar fails by design — run it from a terminal in its folder. A jar that aborts with `java.specification.version=1.8` / "requires a Java Specification 11 VM" means the active JDK is wrong; install JDK 21 and confirm with `java --version` (`java -version` on Windows).
- OSGi env/secret variables are set differently locally than via the `aio` CLI — see [[aemaacs-osgi-configuration]].

## Good for

Components, Sling Models, servlets, dialogs, OSGi configs, dispatcher rules, repoinit scripts, unit + integration tests. **CRX/DE, the OSGi console and local logs are all available** — on no cloud tier are they ([[aemaacs-repository-inspection-by-tier]]).

## Not the cloud runtime

| Missing locally | Where to test it instead |
|---|---|
| Adobe CDN, CDN rules, WAF | Dev/Stage via config pipeline |
| **Assets Compute microservices** (renditions, metadata extraction) | RDE / Dev — local asset processing is a shim |
| Cloud Manager **environment variables and secrets** | RDE / Dev ([[cloud-manager-variables]]) |
| Auto-scaling, multi-pod author, publish farm | Stage |
| Replication service (pub/sub) | RDE / Dev |
| Blue-green reindexing on deploy | RDE / Dev ([[oak-indexing-aemaacs]]) |
| Cloud-managed IMS login, product profiles | RDE / Dev |

Fidelity gap = the reason RDE exists: it runs the real cloud runtime with code pushed in minutes ([[cloud-manager-environments]]).

## Routine

- **Refresh the SDK after every monthly release**, biweekly in practice; new folder, matching jar, same run modes.
- **Dispose of local state daily** so the application never silently depends on stateful data; keep test content in a separate non-shipped package.
- Start with `-Dcom.adobe.granite.crypto.file.disable=true` and **package `/etc/key` once**, install it first on every fresh quickstart, so encrypted OSGi values survive ([[aemaacs-sdk]] §5).
- OSGi configs locally follow the cloud contract — `.cfg.json` under `ui.config` with `config.dev`/`config.author` folders ([[aemaacs-osgi-configuration]]); `dev` run mode is what the quickstart matches.

## Traps

- **"It exists in the quickstart, so I can use it."** A class present at runtime but absent from `aem-sdk-api` is not a supported API; 6.5 code compiled against it fails the Cloud Manager analyser although it runs locally.
- SDK version lagging production: the analyser judges against the latest SDK anyway, so findings change with no code change.
- Encrypted config values typed into a fresh quickstart with no shared `/etc/key` decrypt nowhere else.
- Testing asset workflows locally and declaring them done — renditions are produced by Assets Compute in the cloud, not by the local DAM update workflow.
- Dispatcher "works locally" without the Docker image — only the validator + container reproduce Cloud Manager's checks.
