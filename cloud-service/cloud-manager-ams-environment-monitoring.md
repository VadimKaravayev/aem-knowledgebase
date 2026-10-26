# Cloud Manager environment monitoring + SLA reporting (AMS)

[Adobe page](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-manager/content/using/monitoring-environments) — the **AMS** Cloud Manager doc set, and the feature is AMS-shaped through and through (CSE-owned thresholds, SLA attainment vs contract). AEMaaCS has no equivalent page in Cloud Manager; its observability story is CDN/aemerror logs, Developer Console and the New Relic-less telemetry service. The APM complement on AMS is the bundled New Relic sub-account ([cloud-manager-newrelic-users-api.md](cloud-manager-newrelic-users-api.md)).

## Access and shape

Program ellipsis menu → **Manage → Show Monitoring** → Reports page; requires the **Admin** role. The **System Monitoring** section groups every instance's metrics into four health categories — **Host, Storage, Network, Application** — with hierarchical roll-up: **one critical metric turns its whole category critical** at both overview and instance level (a red "Network" tells you nothing until you drill into which metric). Default view = production instances; others selectable. Detail view per category: time-series graphs **with the thresholds drawn on them**, environment/instance dropdowns, legend-click series filtering.

## Thresholds — the exam-relevant governance fact

Warning and critical thresholds are **defined by Adobe Managed Services and not customer-configurable**; questions about a threshold go to your **CSE**. (Same tell as the pipeline's CSE Oversight gate in [cloud-manager-ams-production-pipelines.md](cloud-manager-ams-production-pipelines.md): a scenario mentioning a CSE = AMS, never AEMaaCS.)

## Metrics by category

- **Host**: Load Per Core (1/5/15 min), Process Count, User Count, Memory Usage %, **JVM Memory (MB)**, **Old Generation Space %** (the GC-death-spiral leading indicator — pairs with `jstat -gcutil` in [jdk-diagnostic-tools.md](../infrastructure-ops/jdk-diagnostic-tools.md)).
- **Storage**: Disk Space per mount (`/`, `/mnt`, …), Folder Size, **AEM Segment Store (GB)** — the repo-growth trend that tells you whether revision cleanup is keeping up ([revision-cleanup.md](../infrastructure-ops/revision-cleanup.md)).
- **Network**: **CQ Port Check** — response time to the author/publish/dispatcher ports; connectivity-as-latency, not throughput.
- **Application**: **Replication Agent** (test-event round-trip seconds per agent) and **Dispatcher Flush** (queue item count) — i.e. the two AMS-replication health signals, the managed-service rendering of the queue checks in [operations-dashboard.md](../infrastructure-ops/operations-dashboard.md) / [monitoring-and-maintaining.md](../infrastructure-ops/monitoring-and-maintaining.md).

## SLA reporting

Production-only comparison against contracted uptime, **separately for Author and End User** (end user ≈ publish/delivery path). Each comes as a triple — **Contract** (the target), **AMS <x> SLA**, and plain **<x> SLA** — monthly attainment graphs with hoverable points, where the AMS variant factors Adobe/vendor-caused incidents separately from the raw measurement, and **scheduled downtime is excluded** from standard SLA math. An **Event Analysis** section lists incidents with time range, cause and comments — the input for "why did we miss the SLA this month" conversations.

Gotchas/limits: thresholds invisible in-product (CSE-only), no customer alert configuration or custom dashboards on this page (alerting = New Relic side), doc states no retention/granularity figures.

Source: Adobe Cloud Manager (AMS) monitoring-environments page. Docs-based, not verified against a live AMS program.
