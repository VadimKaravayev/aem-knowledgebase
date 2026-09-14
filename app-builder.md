# Adobe App Builder (for AEM)

Adobe's sanctioned framework for custom apps, integrations and UI extensions that run **on Adobe infrastructure, outside the AEM JVM**. Not part of AEM — an Experience Cloud extensibility platform that AEM plugs into, alongside Commerce, Analytics, Target. Node.js/React, not Java/OSGi.

The one-line architect answer: **when AEMaaCS won't let you deploy custom infrastructure or heavy custom code, App Builder is where that code goes.**

Docs: [App Builder overview](https://developer.adobe.com/app-builder/docs/overview/), [What is App Builder for AEMaaCS](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/configuring-and-extending/app-builder/extending-aem-with-app-builder), [I/O Runtime system settings/limits](https://developer.adobe.com/runtime/docs/guides/using/system_settings/), [UI Extensibility](https://developer.adobe.com/uix/docs/), [AEM Eventing](https://experienceleague.adobe.com/en/docs/experience-manager-learn/cloud-service/aem-eventing/overview), [Asset Compute](https://experienceleague.adobe.com/en/docs/experience-manager-learn/cloud-service/asset-compute/overview), [API Mesh](https://developer.adobe.com/graphql-mesh-gateway/mesh/), [Extension Manager](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/configuring-and-extending/extension-manager).

There is a [separate 6.5 story](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/implementing/developing/extending-aem/app-builder) — App Builder itself is cloud-side so it works against 6.5 too, but the AEM-side hooks differ (no AEM Eventing, no Asset Compute, no UI extension points; you integrate over HTTP APIs). Assume "App Builder + AEM" means AEMaaCS unless stated.

---

## The stack

| Piece | What it is |
|---|---|
| **Adobe I/O Runtime** | Serverless FaaS on Apache OpenWhisk. Node.js *actions*, stateless, pay-per-invocation. The compute layer, included with App Builder. |
| **Adobe I/O Events** | Event bus. AEM emits state-change events; actions/webhooks/journal consumers subscribe. |
| **API Mesh** | Gateway composing multiple APIs (AEM, Commerce, third-party, private) into **one GraphQL endpoint**, with transforms, WAF and DDoS protection. |
| **React Spectrum + CDN-hosted SPA** | Adobe's design system, so extensions look native inside the Experience Cloud unified shell. |
| **State & Files SDK** | Key-value store and blob storage — actions are stateless with no durable local FS. |
| **Adobe Developer Console** | Projects, workspaces (Stage/Prod), credentials, IMS OAuth. Deployment target for extensions. |
| **`aio` CLI** | Scaffold, run locally, deploy. CI/CD usually GitHub Actions. |

Auth is **IMS-based** throughout and access is gated by Adobe org entitlements — that is the difference from "just run a Lambda".

Adobe's own taxonomy of use cases: **middleware extensibility** (connectors to external systems), **core services extensibility** (custom capability behind Adobe services), **user experience extensibility** (custom UIs and back-office apps).

---

## Recommend it when (decision guide)

AEMaaCS is deliberately closed: no custom infrastructure, immutable `/apps`, Cloud Manager as the only deployment path, quality gates that fail the build, pods you must not starve. App Builder is the escape valve.

| Situation | Why App Builder, not the alternatives |
|---|---|
| **Custom asset renditions on AEMaaCS** | You *cannot* write a custom DAM workflow step calling ImageMagick — the 6.x pattern is gone. The replacement is an **Asset Compute worker** deployed to I/O Runtime and wired in via **Assets Processing Profiles**. Canonical exam scenario. |
| **Third-party integration** (PIM, ERP, CRM, DAM, translation, search) | Vendor SDKs, retries, polling and credential handling stay out of the AEM JVM — AEM stays upgradeable and its request threads stay free. See `aem-sdk-java-ceiling-dual-65-cloud.md` for why shrinking the Java surface is worth real money. |
| **Event-driven reaction to AEM state changes** | AEM Eventing → I/O Events → Runtime action. No polling loop, no scheduled Sling job, no custom `EventHandler` on publish. |
| **Long-running / bulk / bursty work** — migrations, mass imports, report generation | Runs on separately-scaled infrastructure; won't blow Sling job pools or trip Cloud Manager performance gates. Up to 3 h per action vs. a request thread you must not hold. |
| **Custom UI inside the Adobe authoring shell** — buttons, rails, modals, panels | The only supported way to extend the modern AEM UIs. No `/apps` overlay of a Coral/Granite console (those UIs aren't JCR-rendered at all). |
| **Aggregating several backends for one frontend** | API Mesh — one GraphQL call to the browser instead of a BFF servlet on publish. |
| **Back-office app that needs Adobe identity but isn't a website** | Full SPA hosted by Adobe, IMS SSO for free, no AEM page tree abused as an app. |

### Do not recommend when

- The logic is **content-coupled**: rendering, components, templates, authoring dialogs, Sling Models, anything needing JCR/session access or AEM's security context. That is a `ui.apps` / OSGi bundle. App Builder only sees HTTP APIs.
- **Synchronous, latency-critical page-render dependency.** Cold starts exist and blocking invocations are hard-capped at 60 s. Cache or pre-compute instead.
- The need is **header/redirect/edge logic** → `cdn.yaml` rules, then `edge-functions.md`. Edge Functions run at the CDN in ~1 s CPU; App Builder is the *other* end of the spectrum (long, heavy, stateful-ish, event-driven).
- **Client is on 6.5/AMS and wants UI extensions or Asset Compute** — those hooks don't exist there.
- The customer has **no App Builder entitlement** and no budget line for one (see licensing below) — quietly designing an architecture around an unlicensed product is a classic discovery failure.

---

## Where it attaches to AEM

```
                      ┌──────────────── Adobe Developer Console (project/workspace, IMS) ─────┐
                      │                                                                       │
AEM as a Cloud Service│                                                                       │
  ├─ state change ────┼──► Adobe I/O Events ──► Runtime action  ─┐                            │
  │                   │                      ├─► webhook          ├──► your logic ──► AEM HTTP APIs
  │                   │                      └─► journal (pull)  ─┘                    (write back)
  ├─ Processing Profile ─► Asset Compute worker (I/O Runtime) ──► rendition back into the asset
  └─ authoring UI ────┼──► UI extension (React SPA in the shell) ──► Runtime action ──► anything
                      └───────────────────────────────────────────────────────────────────────┘
```

**Three distinct attachment points** — keep them separate in your head, they're often conflated:

1. **AEM Eventing** — AEM produces events, I/O Events fans them out. Subscribe per AEM solution (Sites, Assets). Three consumption styles: **Runtime action** (serverless, push), **webhook** (your own endpoint, push), **journaling** (pull via the Journaling API — use when you need replay or can't expose an endpoint).
2. **Asset Compute** — the AEMaaCS replacement for custom rendition workflow steps. Worker on I/O Runtime, registered through a Processing Profile. Also does **metadata workers**.
3. **UI extensions** — a JavaScript/React app embedded in the Experience Cloud unified shell, talking to the host UI over a **two-way communication protocol**. Deployed to Adobe Developer Console, then enabled per environment in the **Extension Manager** (enable/disable per instance, configure parameters, generate preview links, opt into Adobe's first-party experimental extensions).

### Extensible AEM UIs (verified Sept 2026)

- AEM Experience Hub
- AEM Content Fragments Console
- AEM Content Fragments Editor
- Universal Editor (header menu buttons, properties-panel items, custom events)
- AEM Assets View — **Assets Ultimate only**; extension point `aem/assets/details/1`
- (non-AEM, same framework: Adobe Commerce Admin)

Classic Sites authoring (the Granite/Coral page editor) is **not** in this list — extend it the old way, via `/apps` overlays and clientlibs.

---

## I/O Runtime limits (verified Sept 2026)

These decide feasibility, so quote them from the doc, not memory — they move.

| Limit | Value |
|---|---|
| Action timeout | default **60 s**, max **3 h** (10,800,000 ms) |
| **Blocking** invocation | hard **60 s**, non-negotiable |
| Memory | default **256 MB**, range 128–4096 MB |
| Code size (incl. dependencies) | **22 MB** |
| Parameters / POST payload / action result | **1 MB** each |
| Local storage per action | 600 MB (ephemeral) |
| Concurrency | 100 activations queued per namespace; 200 parallel per container (1–500 configurable) |
| Invocation rate | **6,000 actions/min** (18,000 on request); triggers 600/min |
| Sequence length | 50 actions |
| Log per action | 10 MB stdout |
| Activation record TTL | **7 days**, not configurable |

---

## Gotchas

- **Stateless, and the 1 MB ceiling is the real constraint.** Parameters, POST body *and* the action result are each capped at 1 MB. Never pass binaries or big JSON between actions — write to the **Files SDK** and pass a reference. A bulk export that "works in dev" dies the first time the result crosses 1 MB, and the failure reads as a generic activation error.
- **The 3 h max does not mean you get 3 h synchronously.** Anything a caller waits on is capped at 60 s. Long work must be fire-and-forget: non-blocking invoke, then poll/callback/event. Designing a "long-running" integration as a blocking HTTP call is the most common App Builder design error.
- **Activations expire after 7 days** — logs and results are gone. If you need an audit trail of what an integration did, forward it somewhere yourself; don't treat the activation log as a record.
- **Local storage is ephemeral and per-container.** Warm containers make it *look* persistent in testing. Use State/Files for anything that must survive.
- **Two deployment systems, two release trains.** AEM code goes through Cloud Manager; App Builder goes through `aio`/Developer Console. A change spanning both is not atomic — version your contract (API shape, event payload) and make each side tolerate the other being older. Don't hand a client a release plan that assumes one deploy.
- **Workspaces are not environments.** Developer Console workspaces (Stage/Production) must be mapped deliberately to AEM environments; it's easy to leave a dev workspace's credentials wired into an AEM stage config.
- **UI extensions need enabling per environment** via the Extension Manager after deployment — deployed ≠ visible. Expect "it works in dev" reports that are actually just an unenabled extension.
- **Assets View extensions are Assets Ultimate only.** Check the SKU before promising asset UI work.
- **Cold starts.** Not a place for anything on the critical path of a page render.

## Licensing — ask, don't assume

App Builder is **not automatically included** with an AEM licence. Adobe's own overview routes licensing and trial questions to the FAQ / a sales rep rather than stating an included allocation; some Experience Cloud SKUs bundle an allocation, others don't. **Unverified — confirm entitlement with the client's Adobe account team before designing on it.** In an exam answer, "App Builder, subject to entitlement" is the safe phrasing.

---

## Boundary cheat-sheet

| Need | Home |
|---|---|
| Rendering, components, templates, dialogs, Sling Models, servlets | AEM (`ui.apps` + OSGi bundle) |
| Custom renditions / asset metadata enrichment on AEMaaCS | **App Builder** (Asset Compute worker + Processing Profile) |
| Sync with an external system, scheduled or event-driven | **App Builder** (+ AEM Eventing) |
| Heavy/bursty compute that must not affect AEM performance | **App Builder** |
| Custom UI in the modern AEM consoles / Universal Editor | **App Builder** UI extension (+ Extension Manager) |
| Custom UI in the classic Sites page editor | AEM `/apps` overlay + clientlibs |
| Aggregating several APIs for one frontend | **API Mesh** |
| Request/response manipulation, geo/header personalisation, API key proxying at the CDN | Edge Function — see `edge-functions.md` |
| Static header/redirect/traffic rules | `cdn.yaml` |
| Edge-rendered marketing pages | Edge Delivery Services (different product — don't conflate) |

Adobe docs read **Sept 2026**; Runtime limits and the extensible-UI list verified against the live docs on that date. Licensing specifics not verified.