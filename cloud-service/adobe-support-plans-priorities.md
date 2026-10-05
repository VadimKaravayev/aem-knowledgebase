# Adobe support plans and P1–P4 priorities (Experience Cloud / AEM)

The [enterprise support terms helpx page](https://helpx.adobe.com/support/programs/enterprise-support-terms.html) is bot-blocked; this is assembled from the Experience League **Success Plans data sheets** (overview last updated May 2026) and the prior-plan sheets. Support plans are a different contract layer from the **availability SLA** (Unified SLA + Actionability Addendum, [aemaacs-licensing-content-requests.md](aemaacs-licensing-content-requests.md); AMS SLA attainment reporting in [cloud-manager-ams-environment-monitoring.md](cloud-manager-ams-environment-monitoring.md)) — the SLA promises uptime, the support plan promises **response**.

## The P1–P4 ladder (stable across plans; targets vary)

- **P1** — "production business functions are down or have significant data loss or degradation of services and immediate attention is required." Always **24x7x365**, phone-reachable.
- **P2** — "major service degradations or potential data loss, or a major feature is impacted."
- **P3** — "minor to no degradation … a solution/workaround allows business functions to continue."
- **P4** — "general question regarding current product functionality or an enhancement request."

Initial-response targets are **targets, not SLAs with credits** — and they're *response*, not resolution.

## Current plans (two since ~2023)

| | **Expert Success** (included with every Experience Cloud license) | **Ultimate Success** (paid) |
|---|---|---|
| Team | pooled support engineers | designated **TAM + CSM + Support Services Manager** |
| P1 | 24x7 (phone) | **15 min**, 24x7, prioritized queue |
| P2–P4 | business hours | P2/P3 **24x5**, prioritized |
| Named Support Contacts | standard allotment | **up to 15 per product** |
| Extras | — | Success Accelerators, escalation mgmt |
| Add-ons | Advanced Cloud Support, Priority Queue Bundle | Technical Account Director; Priority Queue Bundle **included** |

(Expert's exact P1 minute-target wasn't retrievable from the blocked datasheet — verify before quoting in a contract discussion.)

## Prior plans (still in older contracts and older exam material)

- **Enterprise Support**: P1 **30 min / 24x7**, P2 **1 h / 24x5**, P3 **2 h / business hours**, P4 **1 day**; **10 Named Support Contacts** per product; a **Named Support Engineer** (designated technical contact in Adobe Support).
- **Elite Support**: P1 **15 min / 24x7**, P2 **30 min / 24x5**, P3 **1 h / 24x5**, P4 1 day; **15 contacts**; **TAM + Named Support Engineer**; per-year allotments of **4 service reviews, 4 expert sessions, 4 field service activities**; event management.
- Also legacy: Business, Online/Standard tiers (lower targets, no named engineer).

## Role disambiguation (exam-grade)

- **Named Support Contact** — *customer-side* person authorized to file/escalate cases (counted per product).
- **Named Support Engineer / TAM** — *Adobe-side*, come with premium support plans (Ultimate today; Elite/Enterprise before).
- **CSE (Customer Success Engineer)** — the **AMS** operations role (pipeline oversight gates, monitoring thresholds, autoscaling/blue-green validation — [cloud-manager-ams-production-pipelines.md](cloud-manager-ams-production-pipelines.md), [cloud-manager-ams-introduction.md](cloud-manager-ams-introduction.md)); not part of a support plan.
- **Development Consultant** — the SI defined in the PSLT with the certified-developer requirement, again a different axis.

A scenario mixing "15-minute P1 response" with "TAM" is describing **Ultimate Success** (or legacy Elite), not a bigger AEM license and not AMS.

Sources: Experience League Success Plans overview + Elite data sheet (May 2026), Enterprise Support datasheet via search; helpx terms page 403s automated fetchers. Oct 2026.
