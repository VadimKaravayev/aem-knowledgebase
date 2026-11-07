# Adobe Experience Platform architecture: layers, stores, edge vs hub, guardrails

Adobe Experience Platform (AEP) is a pipeline: collect → ingest → store → resolve identity → build profiles → evaluate audiences → activate. Real-Time CDP, Journey Optimizer (AJO) and Customer Journey Analytics (CJA) are applications on top of it, and each reads from a different part. Most design mistakes come from mixing up the **two stores** (data lake vs profile store) or the **two locations** (edge vs hub).

Docs-based. Adobe's own architecture pages are mostly diagrams and video. The guardrail numbers below come from Adobe's Profile/Segmentation and Identity guardrail pages and change over time, so check them before quoting one to a client. Contracts and product descriptions can set different limits.

---

## 1. The layers

```
 COLLECT            INGEST                 STORE                    UNIFY                    ACTIVATE
 Web SDK ─┐                            ┌─ data lake ──────────▶ Query Service / Data Distiller, CJA
 Mobile SDK ┼▶ Edge Network ─▶ datastream ┤  (datasets of batches)
 Server API ┘     │                    │
                  │ event forwarding   └─ "enabled for Profile" ─▶ Identity Service ─▶ Profile store
 sources ─────────┼── batch / streaming ─┘                                │ merge policy
 (cloud storage, CRM, Event Hubs, Analytics…)                             ▼
                  ▼                                                Segmentation ─▶ destinations
          third-party endpoints                                      (batch | streaming | edge)   ─▶ AJO
 legend: cross-cutting = sandboxes, XDM, Data Prep, governance (labels + policies), consent, Privacy Service, monitoring
```

| Layer | Components | What it does |
|---|---|---|
| Collection | Web SDK (`alloy.js`), Mobile SDK, Edge Network Server API, Tags, datastreams | Sends XDM (or raw `data` mapped later) to the Edge Network; the **datastream** routes it to AEP, Target, Analytics, AJO, event forwarding |
| Server-side forwarding | Event forwarding (an Edge-type Tags property) | Rules that run on Adobe's edge and send collected data to third-party endpoints, so no third-party script runs in the browser |
| Ingestion | Sources (batch and streaming connectors), Batch Ingestion API, HTTP streaming API, **Data Prep** | Validates against XDM; Data Prep mapping functions transform raw fields into XDM inside the dataflow |
| Model | **XDM**: classes (Individual Profile = attributes, ExperienceEvent = time series, plus lookup/B2B classes), field groups, schemas, **identity namespaces** | The contract every layer reads. Schemas enabled for Profile can only grow; breaking changes are refused |
| Store 1 | **Data lake**: datasets = batches of files | Everything ingested lands here. Read by Query Service, Data Distiller, CJA, exports |
| Store 2 | **Profile store** (Real-Time Customer Profile) | Only datasets **enabled for Profile**. Stores fragments per dataset, merged at read time |
| Identity | **Identity Service**: identity graph per person | Links identities seen together in one record (ECID + CRM ID + email) so fragments become one profile |
| Segmentation | Audiences: batch, streaming, edge | Rule evaluation over profiles; method chosen by the rule's shape |
| Activation | Destinations (streaming, file-based, edge such as Custom Personalization and Target), Destination SDK | Sends audiences and attributes out; governance policies are enforced here |
| Apps | Real-Time CDP, AJO, CJA | RT-CDP = profile + audiences + destinations; AJO = journeys and decisioning on Profile events and audiences; **CJA = analysis on data-lake datasets, not the profile store** |

## 2. Two stores, two purposes

| | Data lake | Profile store |
|---|---|---|
| Contains | every ingested dataset | only Profile-enabled datasets |
| Shape | rows per batch, history kept | fragments per identity, merged at read |
| Access | SQL (Query Service), CJA, dataset export | Profile viewer, audiences, Profile APIs, edge projection |
| Good for | analytics, validation, derived attributes | "who is this person now", activation |

Consequences:
- A dataset not enabled for Profile can still be queried, but **no audience can use it**.
- Deleting or not ingesting a field in the lake does not change what is already in Profile, and the reverse.
- **Data Distiller** closes the loop: SQL over the lake → derived dataset → enable for Profile → new attribute on the profile.

## 3. Edge vs hub

- **Edge Network**: Adobe's globally distributed collection and decision layer. Web SDK, Mobile SDK and Server API calls terminate here. It assigns **ECID**, applies the datastream, runs event forwarding, and serves **edge segmentation** and personalization from a projection of the profile.
- **Hub**: the regional AEP instance with the data lake, Identity Service, the full profile store, streaming and batch segmentation, destinations and AJO.

Data entered at the edge reaches the hub asynchronously. An edge decision uses what the edge projection already has. Only a merge policy flagged **Active-On-Edge** is used there, and edge audiences must be rules simple enough for the edge to evaluate.

## 4. Audience evaluation methods

| Method | Evaluated | Typical latency (Adobe guardrail) | Use for |
|---|---|---|---|
| Batch | scheduled, over all profiles (about daily) | hours | complex rules, aggregates, large populations |
| Streaming | on each incoming event or attribute change at the hub | qualification "may take up to 5 minutes"; 1500 RPS shared across sandboxes (soft) | near-real-time triggers, journeys |
| Edge | on the Edge Network, per request | processing "may take up to 350 ms"; 1500 RPS (soft) | same-page / next-page personalization |

The **rule decides the method**, not the user. Rules that streaming or edge can't evaluate (for example, aggregations over long windows) become batch. "Real-time" use cases fail quietly this way. Audiences are also all re-evaluated in the daily batch run.

## 5. Guardrails worth remembering

**Profile** (Profile and Segmentation guardrails page):

| Limit | Value | Type |
|---|---|---|
| Identities in one person's graph (for Profile use) | 50 | hard: profiles above it are **excluded from segmentation, exports and lookups** |
| ExperienceEvents per profile | 5000 | soft: segmentation uses only the latest 5000 |
| ExperienceEvent size | 10 KB | hard: larger events dropped |
| Profile record size | 100 KB | hard: larger records dropped |
| Profile fragment / stored profile | 50 MB | hard / soft |
| Datasets per class (Individual Profile, ExperienceEvent) enabled for Profile | 20 each | soft |
| Analytics report-suite datasets enabled for Profile | 1 | soft |
| Profile or ExperienceEvent batches per day | 90 | soft |
| Audiences per sandbox: all / streaming (incl. edge) / edge | 4000 / 500 / 150 | soft |
| Audience size | 30% of profiles | soft |

**Identity** (Identity Service guardrails page):

| Limit | Value / behaviour |
|---|---|
| Identities per graph | 50; a full graph drops identities **first in, first out**: cookie IDs first, then device IDs, then cross-device IDs, email, phone; oldest timestamp first within a type. Removing an identity removes its links and can **split the graph** |
| Links to one identity in a single batch | 50 or more → the identity is treated as invalid and its links are filtered from the batch (anti-collapse) |
| Identities per XDM record | 20 |
| ECID value | exactly 38 digits, else the record is skipped |
| Other identity values | ≤ 1024 characters; `null`, `anonymous`, `invalid`, empty are blocked |
| Custom namespaces | no limit |
| Adobe Analytics ID (AAID) | blocked from Identity Service for newer orgs by default |

Placeholder values like `anonymous` or `0` sent as a CRM ID are the classic graph-collapse cause. `null`, `anonymous` and `invalid` are blocked, but anything else that repeats across users (`0`, `unknown`, a shared test email) still collapses graphs.

## 6. Cross-cutting services

- **Sandboxes**: production and development, isolating all config and data. Promote config with **sandbox tooling packages**. Every API call carries `x-sandbox-name`.
- **Governance**: data usage labels on fields and datasets (contract, identity, sensitive) + marketing actions + policies. Enforced at **activation**, so a violating export is blocked before data leaves.
- **Consent**: stored as profile data (consent field groups) and set from Web SDK `setConsent`. It only works if every audience, destination and journey reads it.
- **Privacy Service**: access and delete requests across Platform services and the lake.
- **APIs**: `platform.adobe.io`, OAuth Server-to-Server from the Developer Console; headers `Authorization`, `x-api-key`, `x-gw-ims-org-id`, `x-sandbox-name`.

## 7. Design traps

1. **Email as primary identity.** Shared and changing emails merge different people. Prefer a system-issued person ID as primary and email as secondary.
2. **No guard against shared devices.** Household or kiosk logins link two CRM IDs through one ECID. Use identity graph linking rules (unique namespace, namespace priority) on the person ID.
3. **"Real-time" audiences that are batch.** Check the evaluation method after saving, not the intent.
4. **Expecting CJA to show profile attributes merged by policy.** CJA reads datasets; profile merge policies don't apply.
5. **Enabling everything for Profile.** Counts against the dataset and event guardrails, and the 5000-event window silently trims history for active users. Put analytics-only data in the lake only.
6. **Fixing schemas after go-live.** Profile-enabled schemas only accept additive changes; model in a design document first.
7. **Edge personalization reading attributes the edge doesn't have.** The edge projection and Active-On-Edge merge policy decide what is available.

## What this means for AEM

AEM is a content system next to AEP, not part of it. Typical touchpoints:
- **Collection from AEM Sites**: Core Components emit the **Adobe Client Data Layer** (`adobeDataLayer`); a Tags property maps it to XDM and sends it with Web SDK. AEM itself sends nothing to AEP.
- **Content for activation**: Content Fragments and Experience Fragments exported as **Target offers**; AJO can use AEM content and assets in messages where the integration is configured (check the current AJO docs for the supported path).
- **Personalization on AEM pages**: edge audiences returned by Web SDK and applied client-side, or by Target. The AEM Dispatcher cache stays anonymous; personalization happens after the cached page loads.
- On AEMaaCS nothing changes in this picture. The integration is client-side and through Tags, not an AEM server-side connector.
