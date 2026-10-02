# FileVault packageType taxonomy: application / content / container / mixed

Which `<packageType>` each archetype module declares, what each type may contain, and the trap that makes "where do run-mode OSGi configs go" an exam favorite. The type isn't cosmetic: the FileVault package-type validator (run locally by the `filevault-package-maven-plugin` and again by Cloud Manager) enforces per-type content rules, and on AEMaaCS the type routes the artifact to the right deployment path (immutable image build vs mutable install — see [rolling-deployment-two-version-overlap.md](../cloud-service/rolling-deployment-two-version-overlap.md), where a *mixed* `/apps`+`/conf` package silently installs only its mutable half).

## The taxonomy

| Type | Archetype module | May contain | May NOT contain |
|---|---|---|---|
| `application` | `ui.apps` | immutable `/apps` code: components, HTL, dialogs, clientlibs | **OSGi configs or bundles**, mutable roots |
| `content` | `ui.content` | mutable `/content`, `/conf` (editable templates, CA configs) | anything under `/apps` |
| `container` | `all`, **`ui.config`** | embedded sub-packages, **OSGi bundles and configurations** in install/config folders | regular content |
| `mixed` | — (legacy) | anything | — no validation guarantees; avoid |

The split people get wrong: **mutable vs immutable** only separates `content` from the rest. *Within* immutable, the line is **JCR application content vs OSGi-installer payload** — and installer payload (anything in `install*`/`config*` folders that the JCR installer processes: `.cfg.json` files, embedded jars) belongs exclusively to `container`.

## Why `ui.config` is `container`, not `application`

The intuition "OSGi config is immutable code under `/apps` → `application`" is exactly the planted exam trap. The validator rule kills it: **an `application` package must not contain OSGi configuration or bundles** — a `config.author`/`config.publish.prod` folder inside `ui.apps` fails validation (locally and in Cloud Manager's Build step). So the archetype keeps all run-mode configuration in its own `ui.config` module, `<packageType>container</packageType>`, rooted at `/apps/<app>/osgiconfig` — verified in the archetype's `ui.config/pom.xml` on the `develop` branch, Oct 2026. Run-mode *resolution* semantics (replace-not-merge per PID) are in [osgi-config-runmode-resolution.md](../infrastructure-ops/osgi-config-runmode-resolution.md); assembly into `all` and the embed-folder grammar are in [all-package-embed-structure.md](all-package-embed-structure.md) (which this doc corrects: it previously claimed `ui.config` was `application`).

## Fast elimination grid for exam/config-review

- "Module for run-mode / OSGi configurations" → `ui.config` → **`container`**.
- `content` for configs is wrong twice: configs aren't authorable content, and `/conf` is the context-aware-config world ([context-aware-configuration.md](../sites-content/context-aware-configuration.md)) — a `<PID>.cfg.json` under `/conf` is dead content the OSGi installer never reads.
- `application` for configs fails validation — the "closest wrong answer" by design.
- A real-world package that mixes `/apps` and `/content` either declares `mixed` (and loses validation + deploys unpredictably on AEMaaCS) or should be split. The archetype's four-module layout (`ui.apps`/`ui.config`/`ui.content`/`all`) exists precisely so each artifact has one honest type.

Archetype pom verified via GitHub Oct 2026; validator rules from FileVault/Adobe docs, not re-run locally for this doc.
