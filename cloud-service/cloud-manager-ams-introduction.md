# Cloud Manager for AMS — intro-page facts: autoscaling and blue/green

From the [AMS Cloud Manager introduction](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-manager/content/introduction) ("this documentation specifically describes … Cloud Manager for Adobe Managed Services"; AEMaaCS readers are redirected to the cloud-service doc set). The marketing half — CI/CD "from months/weeks to days/hours", code inspection/performance/security validation before production, self-service UI with role-based permissions, customer-defined KPIs (peak page views/min, expected page-load response time → the pipeline gate in [cloud-manager-ams-program-setup-kpis.md](cloud-manager-ams-program-setup-kpis.md)), open API ([cloud-manager-api-cli-sdks.md](cloud-manager-api-cli-sdks.md)) — is covered in depth elsewhere. Two deployment trigger modes: **automatic** (on events like code commit — the "On Git Changes" trigger in [cloud-manager-ams-production-pipelines.md](cloud-manager-ams-production-pipelines.md)) and **scheduled** time frames ("even outside business hours").

What this page uniquely documents are two **optional, CSE-gated AMS runtime features** people assume don't exist because "AMS is fixed-size":

## Autoscaling (publish/dispatcher only)

Under high load Cloud Manager "automatically provisions additional capacity using its autoscaling feature," adding **one to ten segments of Dispatcher/publishing pairs**. The load-bearing qualifiers:

- **Horizontal, delivery-tier only** — it scales the publish farm; the **Author tier never autoscales on AMS**. Any scenario whose elasticity requirement includes *editor load* is still AEMaaCS-only ([ams-vs-aemaacs.md](ams-vs-aemaacs.md)).
- **Optional/entitlement-gated** ("if entitled", production-only per the program-setup KPI tab) and eligibility goes through your **CSE or Adobe representative** — not a self-service toggle, not a platform default.
- Contrast with AEMaaCS where both tiers scale as a platform property with nothing to provision or ask for.

## Blue/green deployments (premium)

A **premium** AMS feature for reducing deployment downtime: alongside the live (blue) production delivery tier, "a second set of publish and Dispatcher instances (green) is created and used for deployments", then traffic switches. Restrictions, all stated on the page:

- Requires **additional validation by an Adobe CSE** (again: CSE = AMS tell).
- **AWS/Azure-hosted AMS only**, **production environments only**, and **excluded for Assets-only customers**.
- Publish + Dispatcher only — the author tier is not duplicated.

AEMaaCS contrast: rolling blue/green-style deployment is the built-in default there, for every deploy, both tiers ([rolling-deployment-two-version-overlap.md](rolling-deployment-two-version-overlap.md)) — on AMS it's a paid, validated, delivery-tier-only add-on.

## Exam framing

The old flat rule "AMS cannot scale" is one notch too strong — the accurate version: **AMS can autoscale its publish/dispatcher segments (optional, 1–10 segments, prod, CSE-approved) but can never scale Author, and nothing is elastic by default**. Distractor options like "AMS with auto scaling" are wrong when the scenario needs editor-load elasticity, usage-based economics, or scaling as a platform default — not because the feature is fictional.

Source: Adobe AMS Cloud Manager introduction page, Oct 2026. Docs-based.
