# Cloud transition tools — one-page overview and map of the migration docs

"Cloud transition tools" is Adobe's umbrella for moving an **AEM 6.x on-premise or AMS** instance to **AEM as a Cloud Service**. They split into three workstreams — **assessment**, **code refactoring**, **content transfer** — orchestrated from **Cloud Acceleration Manager (CAM)**. This page is the orientation layer; the verified limits and mechanics live in the linked docs.

## Cloud Acceleration Manager (CAM)

Hosted hub in Experience Cloud (no install). It walks a migration project through **Readiness → Implementation → Go-Live**, hosts the Best Practices Analyzer report, issues **Content Transfer Tool extraction keys**, and tracks migration sets. A migration project in CAM is the unit everything else attaches to.

## 1. Assessment — Best Practices Analyzer (BPA)

- A package installed on the **source 6.x** instance (not in the cloud).
- Produces a readiness report in **five categories**: application code to refactor, repository items to relocate, legacy UI to modernise, deployment/configuration issues, replaced or unsupported 6.x features.
- Purpose is to **estimate time and cost**; it assesses and never fixes.
- Detail: [[aemaacs-migration-assessment-refactoring-tools]].

## 2. Code refactoring tools

Run via **`aio-cli-plugin-aem-cloud-service-migration`** (recommended, one CLI for all) or standalone. Output is a refactored project that then ships through a normal **Cloud Manager pipeline** — the tools do not deploy anything.

| Tool | What it converts |
|---|---|
| **Repository Modernizer** | Legacy Maven project → Cloud layout (`ui.apps` immutable code, `ui.content` mutable content, `ui.config` OSGi, `all` container) — see [[aemaacs-content-package-structure]] |
| **Dispatcher Converter** | AMS / on-prem dispatcher config → Cloud dispatcher structure, validated with the SDK Dispatcher Tools — see [[aemaacs-sdk]] |
| **Index Converter** | Custom **Lucene** Oak index definitions (`/apps` or `/oak:index`) → Cloud format; **not `nt:base` indexes**; **ACS Commons Ensure Oak Index definitions are unsupported and must be hand-converted first** — see [[oak-indexing-aemaacs]] |
| **AEM Modernization Tools** | Static → editable templates, Classic UI → Touch UI dialogs, foundation → Core Components |

Detail: [[aemaacs-migration-assessment-refactoring-tools]]. For the AI-assisted route onto Edge Delivery instead of a 1:1 port, see [[experience-modernization-agent]].

## 3. Content transfer — Content Transfer Tool (CTT)

- Migrates repository **content only**: pages, DAM, users and groups (auto-migrated). Never code, never OSGi config.
- **Two phases**: a package on the source **extracts** into an Adobe-hosted **migration set**; CAM **ingests** it into one or **many Cloud environments in parallel**.
- **Differential top-ups** (extraction with overwrite off + ingestion with wipe off) shrink the content freeze — top up often.
- Hard limits: source **≥ 6.3**; **< 750 M nodes**, **≤ 500 GB author / 50 GB publish** segment store, **≤ 25 GB custom Lucene**, ≤ 20 TB File Data Store; **10 sets per project**, **45-day inactivity expiry** per set; `/apps` `/libs` `/home` `/etc` unselectable; region immutable per set.
- **Indexes are not copied, they are rebuilt** — prune and convert them first.
- Detail: [[content-transfer-tool]]. Binary-heavy sources on AMS often pair this with a shared datastore — [[shared-datastore-binaryless-replication]].

## Order of work

1. BPA on the source → size the effort.
2. Refactor code with the tools → fix what they cannot (removed APIs, custom Java) → deploy via Cloud Manager pipeline ([[cloud-manager-pipeline-steps]]).
3. CTT: first full extraction + ingestion early, repeated top-ups, final top-up in the go-live freeze.

Code and content are **separate workstreams** and run in parallel; neither tool touches the other's domain.

## Exam / design traps

- CTT moves content, not code or config; a question about "migrating OSGi configs with CTT" is a trap.
- BPA and the refactoring tools **size or restructure** — they do not rewrite Java that uses removed 6.x APIs. That is manual.
- Index Converter rejects ACS Ensure Oak Index definitions; convert them by hand before running it.
- Migration sets expire after 45 idle days and are capped at 10 per project — plan top-up cadence around that.
- `/etc` content (except Forms-only exceptions) must be relocated by refactoring, since CTT will not select it.
