# Operations Dashboard (AEM 6.5): health reports, maintenance, diagnosis

Tools → Operations on a 6.5/AMS instance — the UI layer over Sling Health Checks, maintenance Sling jobs and the Oak/Granite diagnostic MBeans. [Adobe page](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/sites/administering/operations/operations-dashboard). AEMaaCS contrast: health/diagnostics move to the **Developer Console** and maintenance tasks are Adobe-run with config-as-code only (Tools → Operations is gone there, per [rolling-deployment-two-version-overlap.md](../cloud-service/rolling-deployment-two-version-overlap.md)).

Key URLs: health reports `/libs/granite/operations/content/healthreports/healthreportlist.html`, maintenance `/libs/granite/operations/content/maintenance.html`, diagnosis `/libs/granite/operations/content/diagnosis.html`, index manager `…/diagnosistools/indexManager.html`, System Overview JSON export `/libs/granite/operations/content/systemoverview/export.json`.

## Health reports

Status cards in three states OK/WARN/CRITICAL; thresholds per card via the gear icon. **Individual** checks = one card each; **composite** checks aggregate by `filter.tags` and are OK only when every member is OK — and an individual check carrying a composite's tag (e.g. `security`) **auto-appears under that composite**, no wiring. Every check is also a JMX MBean under `org.apache.sling.healthcheck` (fleet monitoring hooks there, not on the UI). Default execution cadence 60 s.

OOTB checks worth knowing cold (name → what it reads → thresholds):

- **Query Performance** → Oak `QueryStats` MBean `SlowQueries` → warn when present (slow = hit the traversal limit).
- **Query Traversal Limits** → `QueryEngineSettings` → warn if limits are `Integer.MAX_VALUE` **or below 10,000** (someone "fixed" slow queries by raising the limit — the check calls it out in both directions).
- **Observation Queue Length** → warn > 75% of `maxQueueSize`, critical at full — the "repository events backing up" early warning (slow listeners, see sling-oak-observation threads in [thread-dump-analysis.md](thread-dump-analysis.md)).
- **Asynchronous Indexes** → critical if a lane is failing **or last indexed > 2 h ago**, warn 45 min–2 h — the async-lag check behind the accuracy gotcha in [oak-indexing-aemaacs.md](../cloud-service/oak-indexing-aemaacs.md).
- **Large Lucene Indexes** → warn > 1 B docs, critical > 1.5 B.
- **Disk Space** → `FileStoreStats` → warn ratio < 10, critical < 2 (remember revision cleanup aborts below 25% free — [revision-cleanup.md](revision-cleanup.md)).
- **Request Performance** → `granite.request.metrics.timer` → warn 75th percentile > 200 ms, critical > 500 ms (defaults).
- **Replication Queue** → warn when an item's retry count exceeds the agent's `numberOfRetriesAllowed`.
- **Sling Jobs** → critical if queued > `maxNumQueueJobs` (default 1000) or an active job > 1 h old.
- **System Maintenance** → composite over all maintenance tasks; **critical if a task is unconfigured or outside any maintenance window** — i.e. it audits the config, not just execution results.
- Also: Synchronized Clocks (DocumentNodeStore clusters only), Log Errors (ERROR entries present), Scheduler (Quartz job running > 60 s), Security Checks (composite), Active Bundles, Code Cache (Java 7 relic), Resource Search Path Errors (children under `/apps/foundation/components/primary`).

### Custom health check — the two-part recipe

1. OSGi component implementing Sling `HealthCheck` with properties `HealthCheck.NAME`, `HealthCheck.TAGS`, `HealthCheck.MBEAN_NAME`; `execute()` returns `Result`.
2. **It won't show on the dashboard by itself** (unless a composite's tag picks it up): add an `nt:unstructured` node under `/apps/settings/granite/operations/hc` with `sling:resourceType = granite/operations/components/mbean` and `resource = /system/sling/monitoring/mbeans/org/apache/sling/healthcheck/HealthCheck/<MBEAN_NAME>` — and the parent `hc` folder needs **both `sling:configCollectionInherit` and `sling:configPropertyInherit` = true** or the `/libs` defaults vanish (replace-not-merge again, the [context-aware-configuration.md](../sites-content/context-aware-configuration.md) shape). Composites are created as "Apache Sling Composite Health Check" factory configs (`hc.name`, `hc.tags`, `hc.mbean.name`, `filter.tags`) plus the same dashboard node.

## Maintenance tasks and windows

Defaults: **daily 02:00–05:00, weekly Sat 01:00–02:00**, monthly windows since 6.1. Tasks run **sequentially within a window** (why [revision-cleanup.md](revision-cleanup.md) wants its own). Stock assignment: daily = Revision Clean Up + **Lucene Binaries Cleanup** (reclaims the day's index churn from the datastore); weekly = Workflow Purge, **Data Store Garbage Collection**, Audit Log Maintenance; add-on via **+**: **Version Purge** (the task from [monitoring-and-maintaining.md](monitoring-and-maintaining.md)), Project Purge (`com.adobe.cq.projects.purge.Scheduler`), Ad-hoc Task Purge. Since 6.4 a task is **stoppable** — window close or Stop button suspends without losing progress.

Custom task = OSGi service implementing **`org.apache.sling.event.jobs.consumer.JobExecutor`** (not a Runnable/Scheduler) with `granite.maintenance.name`/`.title`, `job.topics = com/adobe/granite/maintenance/job/<Name>`, optional `granite.maintenance.isStoppable`/`.mandatory`; runmode-gate via `ConfigurationPolicy.REQUIRE` + `granite.operations.conditions.runmode`. Deployed → appears in the dashboard to be scheduled.

## Diagnosis tools

- **Log Messages** — an **in-memory appender**, not the log files: default shows ERROR; gear adds loggers (name + level). Zero-filesystem way to watch a package at DEBUG without touching [monitoring-and-maintaining.md](monitoring-and-maintaining.md)'s logger configs — but it starts empty after restart and configures nothing on disk.
- **Request Performance** — slowest requests, but **only** `/content`, `/etc/design` and `.html` requests are captured; default top 20. (Servlet/API slowness is invisible here — that's rlog.jar territory.)
- **Query Performance** — slowest queries from the Oak `QueryStats` MBean; **Explain Query** tab runs XPath/JCR-SQL/SQL2 and reports the **index actually used** + execution plan/time — first stop for "why is this query traversing". (QueryBuilder maps aren't accepted — extract the generated XPath first, or see [acs-aem-tools-explain-query.md](acs-aem-tools-explain-query.md).)
- **Index Manager** — index list/status UI.
- **Download Status ZIP** — configs, bundle list, Sling metrics, optional logs (date-ranged) + thread dumps; what Support means by "send the status zip". **Download Thread Dump / Heap Dump** buttons beside it (heap = full-size file; the jmap caveats in [jdk-diagnostic-tools.md](jdk-diagnostic-tools.md) apply).

## System Overview

One read-only page aggregating health checks, maintenance task states, system (load/disk/heap), instance (version, run modes, startup date), repository (Oak version, **node store + data store type** — fastest way to answer "is this TarMK or Mongo"), distribution + replication agents, workflow/Sling job counts, estimated node counts, online-backup-in-progress flag, indexing status. `export.json` endpoint makes it the cheap fleet-status scrape.

Source: Adobe 6.5 operations-dashboard page, Oct 2026. Docs-based, not reproduced on an instance.
