# Monitoring and maintaining an AEM 6.x instance

The Adobe "Monitoring and Maintaining" page ([6.5](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/implementing/deploying/configuring/monitoring-and-maintaining), [6.4](https://experienceleague.adobe.com/en/docs/experience-manager-64/deploying/configuring/monitoring-and-maintaining)) is the ops grab-bag: log analysis, version purging, replication monitoring, baseline metrics, JVM triage. Everything here is **6.5/AMS only** — it assumes filesystem access to `crx-quickstart/` and classic replication agents, neither of which exists on AEMaaCS (logs via Cloud Manager Download/Tail Log, maintenance is Adobe-internal, publication is journal-based per [content-distribution-journal.md](../cloud-service/content-distribution-journal.md)).

## Log file catalogue (`crx-quickstart/logs/`)

| File | Records |
|---|---|
| `access.log` | every access request to AEM WCM and the repository — **field 3 is the user id**, which is what the authors-count recipes cut on |
| `request.log` | one line per request **and one per response**, paired by request id — the duration source |
| `error.log` | error messages of all severities; the main app log |
| `audit.log` | moderation actions / user modifications |
| `stdout.log` / `stderr.log` | startup event messages / startup errors (stderr default level Warning) |
| `upgrade.log` | upgrade operations (see [in-place-upgrade-6x-to-65.md](in-place-upgrade-6x-to-65.md) — exit code 0 is necessary, this log is the sufficient check) |
| `journal.log` | in `segmentstore/`, TarMK revision journaling — not a text log to read |
| ImageServer / `s7access` | only with Dynamic Media enabled; `s7access` = `/is/image` + `/is/content` requests |

`request.log` format — two lines per request, correlated by the bracketed id, duration on the response line:

```
31/Mar/2009:11:32:57 +0200 [379] -> GET /path/x HTTP/1.1
31/Mar/2009:11:32:57 +0200 [379] <- 200 text/html 33ms
```

Rotation: `error.log` rotates daily to `{name}.yyyy-MM-dd`; **rotated files are never deleted — cleanup is your job** (a full disk from old logs is self-inflicted, and disk pressure also aborts revision cleanup at the <25% threshold, see [revision-cleanup.md](revision-cleanup.md)).

## request.log / access.log analysis recipes

From `crx-quickstart/logs`:

- **Slowest requests** — `rlog.jar`, shipped in `crx-quickstart/opt/helpers/`, sorts request.log by duration, longest first:
  `java -jar ../opt/helpers/rlog.jar -n 10 request.log`
  (`-n` max output lines, `-m` max requests read, `-xdev` excludes CRXDE POSTs — without it a developer's CRXDE saves pollute the top of the list)
- **Total requests**: `grep -R "\->" request.log | wc -l` (count request arrows, not lines — each request is two lines)
- **Distinct authors ever**: `cut -d " " -f 3 access.log | sort -u | wc -l`; on one date: `grep "<date>" access.log | cut -d " " -f 3 | sort -u | wc -l`
- Requests/hour = total requests ÷ elapsed hours between first and last entry

rlog.jar answers "*which* requests are slow"; the next question ("what are those threads doing") is [thread-dump-analysis.md](thread-dump-analysis.md) / [jdk-diagnostic-tools.md](jdk-diagnostic-tools.md). Adobe's tool list for the deeper layer: Apache Bench (`ab`), jconsole, VisualVM, Java Flight Recorder, `truss`/`strace`.

## Baseline metrics (XPath via CRXDE Tools → Query)

Adobe's recommended "know your numbers before they're a problem" queries:

- pages: `//element(*, cq:Page)`
- activations ever: `//element(*, cq:AuditEvent)[@cq:type='Activate']` (÷ days = daily rate)
- MSM rollouts: `//element(*, cq:AuditEvent)[@cq:type='PageRolledOut']`
- live copies: `//element(*, cq:LiveSyncConfig)`
- DAM assets: `/jcr:root/content/dam//element(*, dam:Asset)`
- templates / components: `//element(*, cq:Template)` / `//element(*, cq:Component)`

These are **traversal-heavy on `/`-rooted runs** — fine as occasional baseline collection on author, not as monitoring polls.

## Custom loggers (the two-node contract)

A dedicated log file for your package needs a **pair** of `sling:OsgiConfig` factory configs under `/apps/<project>/config` (runmode rules from [osgi-config-runmode-resolution.md](osgi-config-runmode-resolution.md) apply):

- **Logger** `org.apache.sling.commons.log.LogManager.factory.config-<id>`: `org.apache.sling.commons.log.file` (e.g. `logs/myLogFile.log`), `.names` (String[] of logger/package names), `.level` (`debug|info|warn|error`), `.pattern` (`{0}` timestamp, `{1}` marker, `{2}` thread, `{3}` logger name, `{4}` level, `{5}` message).
- **Writer** `org.apache.sling.commons.log.LogManager.factory.writer-<id>`: same `.file` value (that's the join key), `.file.number` (files kept), `.file.size` (size or date pattern like `'.'yyyy-MM-dd`). Omit the writer and the file uses default rotation.

Global DEBUG = `org.apache.sling.commons.log.level: debug` on `/libs/sling/config/org.apache.sling.commons.log.LogManager` — Adobe's own caution: don't leave it on; prefer a scoped custom logger at debug over the global switch.

## Version purging — and what it is NOT

**Version purge ≠ revision cleanup.** Purging deletes **JCR version history** (the versions authors see in the timeline); revision cleanup ([revision-cleanup.md](revision-cleanup.md)) reclaims **TarMK segment garbage**. They chain: purging versions turns version nodes into garbage that only the next revision cleanup's cleanup phase converts into disk space — so "I purged versions and the disk didn't shrink" is both notes' top non-bug combined.

Two UIs:

- **Scheduled** (the real mechanism): Tools → Operations → Maintenance → Weekly Maintenance Window → Add → **Version Purge** ("Day CQ WCM Version Purge Task"). Dimensions: purge path(s), recursive toggle, **max versions** to keep, **min versions** to keep, **max version age (days)** — empty disables a dimension.
- **Dry run** (classic UI): `/etc/versioning/purge.html` — statuses `ignore (not versionnable)`, `ignore (no version)`, `retained`, `purged`; `V 1.0.1*` = base/current version, never purgeable.

Gotchas: **purged versions are unrecoverable short of a repository restore** — always dry-run first; and Adobe says to define **multiple narrow paths per site** rather than one high root (too many children makes the purge run very long, and it shares the maintenance window with other tasks that run sequentially). Same family: purge workflow instances regularly too.

## Replication agent monitoring

Tools → Replication → Agents on author → click the agent. Check: enabled, target, queue state. Healthy enabled queues read **`idle` or `active`; `blocked` = receiver side** (same diagnostic direction as the cloud journal version — the queue renders where the fault isn't). Actions: View Log, Test Connection, Force Retry, Clear.

**Trap Adobe bolds: never Test Connection on the reverse-replication Outbox on a publish instance** — every item older than the test replication gets reprocessed on every subsequent reverse replication. Agents live under `/etc/replication/author|publish` (query there for fleet-wide monitoring; the curl/JSON endpoints are in [aem-curl-operations.md](aem-curl-operations.md)).

## JVM / resource triage

- **OOM**: grep logs for `java.lang.OutOfMemoryError`, check start-script heap settings; tooling in [jdk-diagnostic-tools.md](jdk-diagnostic-tools.md) (`-XX:+HeapDumpOnOutOfMemoryError` belongs in the start script up front).
- **Disk I/O / thrashing**: Adobe's checklist is *turn off debug collection* in five places — Sling JSP Script Handler, Sling JavaScript Handler, Sling Logging Configuration, CQ HTML Library Manager, CQ WCM Debug Filter — then verify version purging is configured, then "too many open files".
- **Degrades again weeks after every reboot**: memory leak pattern — OOM checks + **unclosed ResourceResolver** hunt.
- **GC visibility**: `-verbose:gc -Xloggc:$LOGS/verbosegc.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps`; JMX for jconsole: `-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=8889 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false` (that's the wide-open dev variant — authenticate/ssl=false is not a production config).
- Adobe's tuning posture since Java 7+: **fixed reasonable heap + JVM defaults is usually right**; exotic GC flag sets are legacy advice.

## Backups (the one rule)

Software installation = stop AEM, back up all of `<cq-installation-dir>`; repository = CRX online backup for hot. Incremental file-datastore backup is supported, but incremental backup of things like Lucene indexes must propagate **deletions**. The load-bearing sentence: **an untested restore means the backup is potentially useless** — test restores, not backups. (Related failure shape: replication ≠ backup, [data-storage.md](../system-design/data-storage.md).)

Source: Adobe 6.5 + 6.4 monitoring-and-maintaining pages. Docs-based, not reproduced on an instance.
