# AEMaaCS maintenance tasks — who owns what, and how purge is configured as code

On AEM as a Cloud Service the **Maintenance card (Tools → Operations → Maintenance) is gone**. Maintenance tasks are either fully Adobe-owned or configured **in source control and deployed via Cloud Manager** — a JCR node override under `/conf` for the Granite purge tasks, or a **config pipeline YAML** for Version Purge and Audit Log Purge. Adobe explicitly **reserves the right to override a customer's maintenance task configuration** to mitigate performance degradation or similar issues.

Source: Adobe docs *Maintenance Tasks in AEM as a Cloud Service*, read Oct 2026 (documented, not reproduced on an environment). 6.5/AMS equivalents: [[operations-dashboard]], [[revision-cleanup]], [[monitoring-and-maintaining]].

## Ownership table

| Task | Owner | Configuration |
|---|---|---|
| **Datastore garbage collection** | Adobe | N/A, fully Adobe owned |
| **Version Purge** | Customer | config pipeline (`MaintenanceTasks` YAML); **disabled by default for now**, will become enabled by default with overridable values |
| **Audit Log Purge** | Customer | same as Version Purge |
| **Lucene Binaries Cleanup** | Adobe | unused, therefore disabled by Adobe |
| **Ad-hoc Task Purge** | Customer | git: window override under `/conf` + child node `granite_TaskPurgeTask` |
| **Workflow Purge** | Customer | git: same window override + child node `granite_WorkflowPurgeTask`; OSGi props per "Regular Purging of Workflow Instances" |
| **Project Purge** | Customer | git: same window override + child node `granite_ProjectPurgeTask`; OSGi props under "Adobe Projects Purge Configuration" |

## Maintenance windows (Granite tasks: Task / Workflow / Project Purge)

Override the OOTB window node from `/libs` by creating a `sling:Folder` at one of:

- Daily — `/conf/global/settings/granite/operations/maintenance/granite_daily`
- Weekly — `/conf/global/settings/granite/operations/maintenance/granite_weekly`
- Monthly — `/conf/global/settings/granite/operations/maintenance/granite_monthly`

Properties (all windows): `windowSchedule` = `daily`|`weekly`|`monthly` (**fixed per node, do not change**), `windowStartTime` / `windowEndTime` as `HH:MM` 24h. **A task runs at most once per window.** Weekly and monthly add `windowScheduleWeekdays="[start,end]"` (two values, 1–7). Monthly adds `windowFirstLastStartDay` = `0` first week of the month, `1` last week; omitted = every month on the weekdays given.

Adobe's three samples, condensed (all carry `sling:configCollectionInherit="true" sling:configPropertyInherit="true"`):

```xml
<!-- daily -->   windowSchedule="daily"   windowStartTime="03:00" windowEndTime="05:00"
<!-- weekly -->  windowSchedule="weekly"  windowScheduleWeekdays="[5,5]" windowStartTime="14:30" windowEndTime="15:30"
<!-- monthly --> windowSchedule="monthly" windowScheduleWeekdays="[5,5]" windowFirstLastStartDay=0 windowStartTime="14:30" windowEndTime="15:30"
```

Then **enable the task** with a child node under that window: name `granite_TaskPurgeTask` / `granite_WorkflowPurgeTask` / `granite_ProjectPurgeTask`, `sling:resourceType="granite/operations/components/maintenance/task"`, `granite.maintenance.name` = `TaskPurge` (etc.). The task's own OSGi properties live in `ui.config` as usual ([[aemaacs-osgi-configuration]]); for Task Purge the component is `com.adobe.granite.taskmanagement.impl.purge.TaskPurgeMaintenanceTask`.

## Version Purge + Audit Log Purge: config pipeline YAML

1. Create e.g. `config/mt.yaml` in the top-level `config` folder of the project.
2. `kind: "MaintenanceTasks"`, `version: "1"`, optional `metadata.envTypes` (e.g. `["dev"]`).
3. `data` holds `versionPurge` and/or `auditLogPurge`. **All properties must be declared — there are no inherited defaults — and types must match.**
4. Deploy with a **config pipeline** in Cloud Manager.

⚠ **Once the `versionPurge` node has been deployed it must stay declared; removing it makes the config pipeline fail.**

Adobe's sample:

```yaml
kind: "MaintenanceTasks"
version: "1"
metadata:
  envTypes: ["dev"]
data:
  versionPurge:
    maximumVersions: 15
    maximumAgeDays: 20
    paths: ["/content"]
    minimumVersions: 1
    retainLabelledVersions: false
  auditLogPurge:
    rules:
      - replication:
          maximumAgeDays: 15
          contentPath: "/content"
          types: ["Activate", "Deactivate", "Delete", "Test", "Reverse", "Internal Poll"]
      - pages:
          maximumAgeDays: 15
          contentPath: "/content"
          types: ["PageCreated", "PageModified", "PageMoved", "PageDeleted", "VersionCreated", "PageRestored", "PageValid", "PageInvalid"]
      - dam:
          maximumAgeDays: 15
          contentPath: "/content"
          types: ["ASSET_EXPIRING", "METADATA_UPDATED", "ASSET_EXPIRED", "ASSET_REMOVED", "RESTORED", "ASSET_MOVED", "ASSET_VIEWED", "PROJECT_VIEWED", "PUBLISHED_EXTERNAL", "COLLECTION_VIEWED", "VERSIONED", "ADDED_COMMENT", "RENDITION_UPDATED", "ACCEPTED", "DOWNLOADED", "SUBASSET_UPDATED", "SUBASSET_REMOVED", "ASSET_CREATED", "ASSET_SHARED", "RENDITION_REMOVED", "ASSET_PUBLISHED", "ORIGINAL_UPDATED", "RENDITION_DOWNLOADED", "REJECTED"]
```

### `versionPurge` properties

| Property | Type | Meaning |
|---|---|---|
| `paths` | string[] | **only `"/content"` is allowed** |
| `maximumAgeDays` | int | versions older than this are removed; `0` disables age-based purge |
| `maximumVersions` | int | versions beyond the n-th newest are removed; `0` disables count-based purge |
| `minimumVersions` | int ≥ 1 | always keep at least this many |
| `retainLabelledVersions` | boolean | **recommended `false`**: every Assets version gets an auto-generated label, so `true` effectively disables purging for DAM. Unlabelled Sites pages are unaffected. (The page's table misspells it `retainLabelledVersioned`; the YAML spelling is the real one.) |

Interaction: `maximumAgeDays=30, maximumVersions=10, minimumVersions=2` → a version goes when there are more than 10. With `maximumVersions=0` nothing newer than 30 days is purged, and one version older than 30 days is kept (the minimum).

**Defaults** once default purging is switched on: remove versions older than **30 days**, keep the **5 most recent** within 30 days, always keep the most recent version + current file. **Environments created before that switch** get the conservative policy: remove only versions older than **7 years (2557 days)**, keep everything within 7 years — Adobe recommends lowering it.

**AEM Guides customers must not configure Version Purge.**

### `auditLogPurge` properties

- `rules`: one or more of `replication`, `pages`, `dam`; **each rule needs all its properties**.
- `maximumAgeDays` (int), `contentPath` (**must be `"/content"`**), `types` (string[] from the enumerations in the sample above; the pages table also lists `PageRolled Out`, which the sample YAML omits).
- **Defaults** after default purging is enabled: replication, DAM and page logs older than **7 days** removed, all events logged. Pre-switch environments: **7 years (2557 days)**, lower it.
- Regulatory need for immutable audit logs → use a specialised external service; AEM's audit log is not that.

## Exam / design traps

- "Open the Maintenance card and add a Version Purge task" — the card does not exist on AEMaaCS; the answer is a config pipeline YAML (purge) or a `/conf` window node in git (Granite tasks).
- "Set `retainLabelledVersions: true` to be safe" — on Assets that silently disables purging; Adobe recommends `false`.
- "Purge versions under `/content/dam` only" — `paths` accepts exactly `"/content"`.
- Partial YAML "to override one value" — no inherited defaults, every property must be present or the config is invalid.
- Removing the `versionPurge` block to "turn it off" — fails the config pipeline; set `maximumAgeDays: 0` and `maximumVersions: 0` instead.
- Datastore GC and revision cleanup questions on AEMaaCS — Adobe-owned, nothing to configure; those answers belong to 6.5/AMS ([[revision-cleanup]]).
- Long-lived pre-switch environments keep 7-year retention until someone declares the YAML.

## References

- Adobe docs — Maintenance Tasks in AEM as a Cloud Service: https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/operations/maintenance
