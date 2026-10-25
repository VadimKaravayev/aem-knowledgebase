# Local development environment for AEMaaCS — the SDK tier below RDE

The local environment is the **SDK Quickstart jar on your machine**, plus optional Dispatcher tooling: the fastest and cheapest tier before an RDE or a pipeline ([[environment-tiers-dev-stage-prod]]). Orientation page; artifact sources, version discipline and the CryptoSupport recipe are in [[aemaacs-sdk]].

## Components

| Piece | What it is | Notes |
|---|---|---|
| **Quickstart jar** | the AEM runtime, author *or* publish | Software Distribution Portal. Rename to `aem-author-p4502.jar` / `aem-publish-p4503.jar` — **the filename sets run mode and port** — then `java -jar`. Recent SDKs need **Java 17+** ([[aem-sdk-java-ceiling-dual-65-cloud]]) |
| **`aem-sdk-api`** | compile-time API jar, successor of the 6.5 uber-jar, `provided` scope | **version must match production AEM** (About dialog / Cloud Manager env card) |
| **Dispatcher Tools** | validator for the dispatcher module + Docker image running Apache + Dispatcher in front of local publish, hot-reloads config | same SDK zip; what Cloud Manager's Build step validates against ([[dispatcher-fundamentals]], [[dispatcher-converter]]) |
| **Build** | `mvn clean install -PautoInstallSinglePackage` deploys the `all` package via Package Manager | runs the **same AEM Analyser** as Cloud Manager, so a clean local build catches most Build Images failures ([[aem-analyser-maven-plugin]], [[cloud-manager-pipeline-steps]]) |

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
