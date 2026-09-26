# AEM as a Cloud Service: architecture overview (Adobe's four views)

Adobe's introductory architecture page describes AEMaaCS from four angles:
**logical** (programs, environments, solutions), **service** (the composable
services and tiers), **system** (how the tiers physically run), and
**development** (repositories and pipelines). This note condenses all four and
records the statements that exam questions and design reviews turn on.

Companion notes: [[ams-vs-aemaacs]] for why AMS is not the same thing,
[[content-distribution-journal]] for the publication mechanism in depth,
[[rolling-deployment-two-version-overlap]] for the deployment consequence,
[[cloud-manager-pipeline-steps]] for what a pipeline actually runs.

---

## 1. Logical architecture: tenant, program, environment

```
IMS org (tenant)
 └─ Program (per licensing entitlement; Sites / Assets / Forms / EDS + add-ons)
     ├─ Production   ─┐ 1:1, same sizing
     ├─ Stage        ─┘
     ├─ Development  (1..n, smaller sizing)
     └─ RDE          (no pipeline, direct push)
```

- A **tenant** is an IMS organisation. It can hold as many programs as it is
  licensed for. Common layout: one central Assets program, several Sites
  programs, one per online experience.
- A **program** is where you name the application, configure it and allocate
  permissions. Edge Delivery Services show up as a top-level solution in Cloud
  Manager but are licensed as part of Sites, Assets or Forms.
- **Four environment types**, and what each is for:

| Environment | Sizing | Purpose | Pipeline? |
|---|---|---|---|
| Production | full | live experiences and business users | yes |
| Stage | **same as production** | automated tests before prod, performance and security tests, content kept in sync via content copy | yes |
| Development | smaller | implement and test under the same runtime; **not** for performance or security tests | yes, same quality gates as prod |
| RDE | dev-class | fast iteration, code pushed without a formal pipeline | no |

Two rules fall out of this table that questions like to probe: run load and
security tests on **stage**, never dev, because only stage matches production
sizing. And the stage-to-production coupling is 1:1.

## 2. Service architecture: two halves under one CDN

```
      CONTENT MANAGEMENT                EXPERIENCE DELIVERY
 ┌──────────────────────┐   Replication  ┌───────────────────────┐
 │ AEM Author tier      │───────────────▶│ AEM Publish tier      │──┐
 │  (page/Universal/CF  │  (pub/sub      │  (publishers +        │  │  Adobe-
 │   editors; Oak repo) │   queues)      │   dispatchers)        │  ├─ managed
 ├──────────────────────┤                ├───────────────────────┤  │  CDN
 │ Document-based       │───────────────▶│ Edge Delivery Publish │──┤
 │  (Word/Excel/GDocs)  │                │  (serverless -> HTML) │  │
 └──────────────────────┘                ├───────────────────────┤  │
                                         │ Edge Delivery Assets  │──┘
                                         └───────────────────────┘
        Preview tiers exist for both publish tiers (single node, may have downtime).
```

Both publish tiers are **origins behind the same Adobe-managed CDN**, and they
are not mutually exclusive: one domain can serve some pages from Edge Delivery
and the rest from AEM Publish. Assets-only programs have **no publish and no
preview tier** by default.

**Adjacent services**, each of which is a plausible answer to a "which service
is responsible for..." question:

| Service | Owns |
|---|---|
| Replication | Processing publish operations and pushing content to the publish tiers. **Redesigned**: not the 6.x replication framework, but pub/sub with cloud queues so any number of publishers can subscribe. This is what makes publish autoscaling possible. |
| Content Repository | The author tier's Oak repository, blob-based cloud storage for binaries. |
| CI/CD | The Cloud Manager subset that runs deployment pipelines. |
| Testing | Infrastructure that executes functional tests, UI tests (Selenium, Cypress) and experience audits (Lighthouse) inside a pipeline, or on an EDS pull request. |
| Data | Exposes licensing metrics (content requests, storage, users) and usage reports through APIs and the Cloud Manager UI. |
| Operational Telemetry | Collects page views, Core Web Vitals, conversion events and answers queries over them. |
| Assets Compute | Processes uploaded images, video and documents into renditions and AI metadata, with access to Photoshop and Lightroom APIs. Runs outside the AEM JVM. |
| IMS | Authentication and user/group management for every Experience Cloud application, administered in the Adobe Admin Console. |

## 3. System architecture: pods, not servers

- Author and Publish are **Docker containers under a container orchestration
  service**. Pod count varies with authoring activity and delivery traffic.
- **Author** is a cluster of pods sharing **one** repository. Minimum two pods
  so maintenance and deployments do not interrupt authors.
- **Publish** is a farm where **each publisher has its own repository** of
  published content and is paired 1:1 with an Apache plus dispatcher instance
  that is the CDN origin. Minimum two pods, commonly more under load.
- **Preview** is a **single node** used for content QA before publishing.
  Downtime during deployments is expected there.
- **Edge Delivery Services** run on CDN plus serverless: on request, serverless
  code converts published content from the author tier or document source into
  semantic HTML and acts as the CDN origin.

The author-shared versus publish-per-pod repository split explains several
behaviours: why publish content is eventually consistent across pods, why the
author tier never needs to know how many publishers exist, and why there is no
reverse replication (see [[ugc-external-database-hosting]]).

## 4. Development architecture: repositories and the one deployment path

Five Cloud Manager repository types:

| Repository | Holds | Deploys to |
|---|---|---|
| Full stack | Java, OSGi config, content packages | author and publish |
| Front end | client-side JS, CSS, HTML (clientlibs) | author and publish |
| Web tier | dispatcher configuration | publish |
| Configuration | CDN settings, maintenance task settings, similar `.yaml` | AEM publish and EDS publish |
| Edge delivery | JS, CSS, HTML for EDS sites (a GitHub repository) | EDS |

**Cloud Manager is mandatory**. It is the only way to build, test and deploy to
author, preview and publish. A pipeline run combines the latest customer
packages with the latest Adobe baseline image to produce a new application
version for both tiers. The same pipeline, with the same Adobe-contributed and
customer-contributed tests, runs whether the trigger is your code change or an
Adobe maintenance release. Tests run on **stage**, which is the second reason
stage content should mirror production.

Cut-over is a **rolling update** of all service nodes, so neither author nor
publish has downtime. The trade-off is that old and new code briefly serve
traffic at the same time, covered in [[rolling-deployment-two-version-overlap]].

## 5. The four innovations since 6.x, and what each implies

| Innovation | Consequence you design around |
|---|---|
| Binaries go **directly** to and from the cloud data store, never through the AEM JVM | Smaller pods, faster autoscaling, faster uploads and downloads. Do not write code that streams large binaries through a servlet when a direct URL exists. |
| Publishing is a **pub/sub pipeline** with queues that publish pods subscribe to | Author is decoupled from publisher count. Blocked queues are the failure mode to diagnose, not stuck replication agents. See [[content-distribution-journal]]. |
| Code and configuration are **immutable, baked into the image** | Every node is identical. `/apps` and `/libs` change only via a pipeline, never via Package Manager or CRXDE. See [[repoinit-acls-on-apps-and-libs]] and [[aemaacs-repository-inspection-by-tier]]. |
| Micro-services on **serverless** technology, notably Adobe I/O Runtime | Heavy or non-JVM work belongs in Assets Compute, App Builder or Edge Functions. See [[app-builder]] and [[edge-functions]]. |

## Exam and review checklist

- Elasticity requirement in a scenario means AEMaaCS, never AMS, because only
  AEMaaCS autoscales compute.
- Performance or security testing belongs on stage. Dev is undersized by design.
- A package that mixes `/apps` with `/conf` or `/content` installs "OK" via
  Package Manager on AEMaaCS but silently drops the immutable half. Split into
  code and content packages.
- Stale content on some publish pods but not others points at the distribution
  queues, not at a dispatcher flush agent, which does not exist in this model.
- "Which component sits between authoring and delivery" is the Replication
  Service. "Which component processes renditions" is Assets Compute.

## References
- [An Introduction to the Architecture of AEM as a Cloud Service (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/overview/architecture)
- [Cloud Manager Repositories (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/managing-code/cloud-manager-repositories)
- [AEM as a Cloud Service Development Guidelines (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/developing/development-guidelines)
