# AEP data activation and Journey Optimizer

How customer data in Adobe Experience Platform (AEP) is turned into experiences: profile lookup, audience export, and journey orchestration with Adobe Journey Optimizer (AJO), plus the analysis tools that close the loop. The platform internals (stores, edge vs hub, identity, guardrails) are in [[aep-architecture-overview]]; this doc starts where that one ends.

Docs-based. Product capabilities and integration paths change often; check the current Adobe docs before committing a design to a specific destination or AJO feature.

---

## 1. Why it exists

Websites, apps, email, SMS, ads, call centres, POS and CRM each keep their own customer record and know little about interactions in the other channels, so experiences are fragmented. A **Customer Data Platform (CDP)** unifies those records into one profile and makes it usable everywhere.

- **AEP** = the data infrastructure (collection, XDM, data lake, Identity Service, Real-Time Customer Profile).
- **Real-Time CDP** = the application on AEP for audience building, segmentation and activation to destinations.
- **AJO** = the application on AEP for one-to-one journeys and decisioning.
- **Adobe Target** = on-site / in-app personalization and testing, fed by AEP profile and audiences.
- **CJA** = analysis of the journeys, reading data-lake datasets.

A profile has three parts: **attributes** (name, preferences, loyalty tier), **ExperienceEvents** (page views, cart adds, purchases, email interactions) and **identities** (email, CRM ID, ECID) linked in the identity graph.

## 2. Three ways to activate

| Mechanism | Pattern | Typical consumer |
|---|---|---|
| **Profile lookup** (Profile API / edge profile) | pull one person's unified profile on demand | call-centre desktop, POS, support app, server-side personalization |
| **Audience activation** | push *who qualifies* to a destination | Target, ad platforms, ESPs, cloud storage / enterprise systems |
| **Journey activation** (AJO) | react to *this person's* event or qualification and orchestrate steps over time | email, SMS, push, in-app, web, code-based channels, custom actions |

Choose by the question being answered: "what do we know about this person right now?" → lookup; "who should see this campaign?" → audience; "what should happen next for this person after X?" → journey.

### Profile lookup
The Profile Access API (`/data/core/ups/access/entities`) reads a merged profile by identity under a chosen merge policy. It is a per-person lookup, not a bulk export; bulk use belongs in dataset export or file destinations. The returned view reflects what is in the profile store, so attributes from datasets not enabled for Profile are absent.

### Audience activation
An audience (segment) is a rule over profile attributes, events and other audiences, e.g. "viewed product X in the last 7 days and did not purchase". Evaluation is batch, streaming or edge depending on the rule's shape (see [[aep-architecture-overview]]).

Destination types:
- **Personalization** (Target, Custom Personalization): audience membership available at the edge for same-page / next-page decisions.
- **Advertising** (social, search, display): usually hashed identifiers matched by the platform; match rate, not profile count, decides reach.
- **Marketing systems** (ESPs, campaign tools) and **enterprise / cloud storage** destinations (file exports to cloud storage, streaming to event hubs and HTTP endpoints).

Latency, identity requirements and refresh cadence differ per destination: a streaming destination does not make a batch-evaluated audience real time, and file destinations export on a schedule. Governance labels and policies are enforced at this step.

## 3. Adobe Journey Optimizer

AJO runs on the same profile, events and audiences, and adds orchestration, messaging and decisioning.

### Journey entry types
- **Unitary event**: one person enters when a configured event arrives (account created, cart add, purchase). Lowest latency; this is the real-time path.
- **Audience qualification**: enters when a person qualifies for or exits a streaming (or batch) audience.
- **Read audience**: a scheduled batch entry of everyone in an audience, for campaign-style sends.

Abandoned cart, the canonical example: cart-add event enters the journey → **wait** (e.g. 2 h) → **condition** checks for a purchase event since entry → if none, send a personalised reminder → branch on open / click / purchase. The purchase check is what makes it a journey rather than a blast; without an exit or condition on purchase the reminder goes to buyers too.

### External data and custom actions
- **Custom actions** call external REST endpoints from a journey step (CRM, loyalty, inventory, pricing, ML scoring). Their **response payload** can be used in later conditions and personalization without being written to the data lake.
- **Data sources** fetch external fields at runtime for conditions and personalization.
- Every external call sits on the journey's hot path: the endpoint must handle the journey's throughput, and AJO applies capping/throttling configuration and timeouts. Design the endpoint for the peak entry rate, not the average.

### Content
Reusable **templates** and **fragments**, profile-based personalization expressions, and offers. AJO integrates with **AEM Assets** (assets selected from AEM inside the AJO content editor) and can use AEM Content Fragments in supported experiences.

### Decisioning
AJO's decisioning picks which offer or content item a person gets from a catalog, using:
- eligibility rules (audience membership, profile attributes, context),
- priority and capping,
- ranking (formulas or AI models, e.g. trained on propensity).

Offers are stored in the decisioning catalog, **not on the profile**; profile data and audience membership are inputs. Reusing the same offer definitions in other Adobe products depends on the specific integration. Adobe has two generations here (the older Offer Decisioning / decision management and the newer Decisioning with decision policies in code-based and message experiences); check which one a sandbox uses before designing.

## 4. Ingestion inputs and analysis

- **Sources and APIs**: batch (cloud storage, databases, CRM connectors) and streaming (HTTP streaming API, Azure Event Hubs and other stream connectors, Web/Mobile SDK via Edge Network).
- **Query Service**: SQL over data-lake datasets for validating ingestion, joining datasets and investigating data quality. **Data Distiller** adds scheduled, persisted transformations whose output datasets can be enabled for Profile as derived attributes.
- **Intelligent Services**: **Customer AI** produces propensity scores (conversion, churn) written to the profile for audiences and decisioning; **Attribution AI** estimates each touchpoint's contribution to conversions.
- **Customer Journey Analytics (CJA)**: cross-channel analysis of journeys (paths, fallout, contribution) on data-lake datasets. AJO orchestrates interactions; CJA analyses them.

## 5. End-to-end flow

1. Web SDK sends interaction events through the Edge Network.
2. The datastream routes them to AEP (data lake) and to Target / AJO as configured.
3. If the dataset is enabled for Profile, the events update the person's profile under the resolved identity.
4. Audiences are re-evaluated (edge, streaming or batch, by rule).
5. Target uses edge profile and audiences to personalise the current or next page.
6. AJO journeys enter on the event or on audience qualification.
7. Decisioning picks the offer or content.
8. Messages go out on AJO channels; audiences go to RT-CDP destinations.
9. Query Service and CJA measure the outcome.

## 6. Traps

1. **Using the Profile API as a bulk export.** It is a per-identity lookup; use dataset export or file destinations for bulk.
2. **Expecting a destination to be faster than the audience.** Activation latency = evaluation method + destination cadence.
3. **Journey triggered by a batch audience for a "real-time" use case.** Use a unitary event entry for anything that must react within seconds or minutes.
4. **No exit criteria.** Without a purchase check or journey exit, abandoned-cart reminders reach people who already bought.
5. **Custom action endpoint sized for average load.** A read-audience entry fires a burst equal to the audience size.
6. **Storing offers as profile attributes.** Offers belong in the decisioning catalog; the profile only holds the inputs.
7. **Analysing journeys with profile merge semantics.** CJA reads datasets, not merged profiles.

## What this means for AEM

- **AEM Assets in AJO**: marketers pick approved assets from AEM inside the AJO editor, so DAM stays the single source for imagery; this requires the AEM Assets ↔ AJO integration to be provisioned for the org (AEMaaCS Assets).
- **Content Fragments** can feed supported AJO experiences, and CF/XF can be exported as Target offers, keeping copy authored in AEM rather than duplicated in each channel tool.
- AEM Sites is a data source only through the client-side path (Adobe Client Data Layer → Tags → Web SDK); AEM itself does not call AEP or AJO server-side.
