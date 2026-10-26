# Assets activity history (Timeline) and where audit events live

[Adobe page](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/assets/manage/assets-activity-history) (AEMaaCS). The asset **Timeline** (select asset → **Timeline** in the GlobalNav rail) is the per-asset activity stream — the audit view, distinct from **version history** (content snapshots you can revert to, [manage-digital-assets-operations.md](manage-digital-assets-operations.md)): timeline answers *who did what when*, versions answer *what did it contain*.

**Tracked operations:** create, delete, **download (incl. renditions)**, publish, unpublish, approve, reject, move — plus uploads/modifications made through **Adobe Asset Link and the AEM desktop app** (remote edits are not invisible).

## Storage — three `/var/audit` buckets, keyed by event family

| Event family | Audit node path |
|---|---|
| standard DAM operations | `/var/audit/com.day.cq.dam/content/dam` |
| publish/unpublish | `/var/audit/com.day.cq.replication/content` |
| move | `/var/audit/com.day.cq.wcm.core.page` |

Useful two ways: programmatic reporting queries `cq:AuditEvent` nodes under the right bucket (the same node type the baseline-metrics queries count — [monitoring-and-maintaining.md](../infrastructure-ops/monitoring-and-maintaining.md)), and a "publish missing from timeline" puzzle may just mean you're looking in the DAM bucket while the event sits under the replication path. `/var` is mutable on AEMaaCS, so these accumulate at runtime — retention is whatever **Audit Log Maintenance** (weekly maintenance task, [operations-dashboard.md](../infrastructure-ops/operations-dashboard.md) on 6.5; Adobe-run on AEMaaCS) purges; the page itself states no retention policy.

## The one limitation that surprises people

**"Transient workflows are not displayed in the timeline, because no history information is saved for these workflows."** On AEMaaCS asset processing is largely transient-workflow/Assets-Compute based — so "why doesn't the timeline show processing steps" is by design, not a bug; and conversely, making a workflow transient (the performance best practice) trades away its timeline/audit trail. Pick per workflow: auditability vs repository churn.

Docs-based. Gaps the page leaves open (retention numbers, event customization, read-ACL on audit nodes): unspecified — verify on an instance before promising an auditor.
