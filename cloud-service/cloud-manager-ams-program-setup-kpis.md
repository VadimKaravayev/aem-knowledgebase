# Cloud Manager for AEM 6.x (AMS): program setup and KPIs

This is the **AMS / AEM 6.x** Cloud Manager, not AEMaaCS. Here a program is
"set up" rather than "created", there is no sandbox/production split, and the
distinctive artefact is the **KPI tab**, whose numbers feed the pipeline's
performance test gate. Compare [[cloud-manager-program-types]] for the
AEMaaCS program model and [[ams-vs-aemaacs]] for the platform difference.

---

## Setup Program: three tabs

Business lead logs into `my.cloudmanager.adobe.com` → **Setup Program**.

| Tab | What goes in |
|---|---|
| **General** | description, optional thumbnail |
| **KPI** | per-licensed-product KPIs (separate Sites and Assets KPIs) used by the performance test step |
| **Provisioning** | on-demand (auto)scaling options, **only if autoscaling is enabled for the program**; autoscaling applies to **production only** and is unavailable for some customers |

Save → provisioning takes several minutes before the program is usable.

**Edit program** later from the Overview page: changes are saved immediately
in Cloud Manager but **only reach the environments on the next pipeline run**.
The action bar switches between programs without going back to the overview.

## How KPIs are measured

- **Sites KPIs run against the staging environment**, so you scale the
  production target down to stage capacity. Adobe's example: 1000 page
  views/minute in production across **four** dispatcher/publish pairs becomes
  a **250 page views/minute** KPI when stage has **one** pair.
- **Assets KPIs**: the test repeatedly uploads assets for **30 minutes** and
  measures per-asset processing time plus system-level metrics.
- **CDN in front (Akamai, CloudFront)**: Cloud Manager hits stage directly,
  so the KPI should reflect only the traffic that would **miss the CDN
  cache**, typically a small fraction of total production traffic. Putting
  the full CDN-absorbed number in the KPI guarantees a failed performance
  gate.

## Exam checklist

- KPIs live in program setup and are the thresholds for the **performance
  test** pipeline step, measured on **stage**, not production.
- Divide production throughput by the stage/production publish ratio.
- Subtract CDN-cached traffic; KPI = expected origin (cache-miss) load.
- Edits to the program apply at the **next pipeline run**.
- Autoscaling settings: production only, and only when entitled.

## References
- [Program setup, Cloud Manager for AEM 6.x (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-manager/content/getting-started/program-setup)
