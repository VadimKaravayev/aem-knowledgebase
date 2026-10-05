# Triage playbook for critical AEM 6.x issues (Adobe KCS ka-17457)

[KB article ka-17457](https://experienceleague.adobe.com/en/docs/experience-cloud-kcs/kbarticles/ka-17457) — "How to analyze common critical AEM issues": Adobe Support's own first-response checklist across six failure categories. **Era warning:** the article is 6.2-vintage (JDK 7/8 talk, offline compaction schedules, pre-6.3 `oak.plugins.segment` package names) — the *method* holds on 6.5, but cross-check maintenance advice against [revision-cleanup.md](revision-cleanup.md) before acting on it.

## 1–2. Sites / Assets performance

The four-step frame (Assets explicitly reuses Sites' steps 1–4):

1. **Series of thread dumps** → [thread-dump-analysis.md](thread-dump-analysis.md) / [jdk-diagnostic-tools.md](jdk-diagnostic-tools.md).
2. **OS-level CPU** (`top` / Task Manager) — correlate hot threads before blaming AEM.
3. **request.log for slow requests** → rlog.jar recipes in [monitoring-and-maintaining.md](monitoring-and-maintaining.md).
4. **Maintenance hygiene review** — the article's schedule table is where its age shows: "Revision Clean Up daily (MongoMK)", "**Offline Tar Compaction bi-weekly** (TarMK)" is pre-6.3 advice — since 6.3 Online Revision Cleanup is the default daily task and offline compaction is migration-only ([revision-cleanup.md](revision-cleanup.md)); DSGC/workflow/version/audit purges weekly still stand ([operations-dashboard.md](operations-dashboard.md)).

Plus dispatcher caching review and client-side audits (Chrome DevTools) — i.e. rule out "it's not the server at all" early.

## 3. Memory

- Console: **`/system/console/memoryusage`** (6.x only; also triggers heap dumps).
- Watch **Old/Tenured Generation**, not total heap. The article's heuristic: **"on Oak Tar storage, tenured usage higher than 3 GB might be a problem"** — a rough 6.2-era flag, useful as "TarMK itself shouldn't need huge tenured residency; a fat old gen = application caches or a leak", not a hard limit on today's heaps.
- High CPU attributed to **"VM Thread" or GC threads = a memory problem wearing a CPU costume** — same signal as `jstat -gcutil` old-gen ~100% in [jdk-diagnostic-tools.md](jdk-diagnostic-tools.md).
- Heap dumps → Eclipse MAT **Leak Suspects report** + Thread Details view.

## 4. Indexing ("search results stale/missing")

Go straight to the async indexer MBeans in the JMX console:

- `org.apache.jackrabbit.oak:name=async,type=IndexStats`
- `org.apache.jackrabbit.oak:name=fulltext-async,type=IndexStats` (6.2+; one bean per lane)

Three fields answer it: **`FailingSince`** (when it broke), **`LastError`** (the stack trace — read it before anything else), **`LastIndexedTime`** (**> 5 min old = indexing too slow** per the article; the 6.5 Operations health check warns at 45 min / goes critical at 2 h — [operations-dashboard.md](operations-dashboard.md) — so 5 min is the "users notice" line, not the alarm line). Mechanism background: [oak-indexing-aemaacs.md](../cloud-service/oak-indexing-aemaacs.md).

## 5. Replication (6.x classic agents)

Queue states Active / Idle / **Blocked** (queued but can't process → receiver side, same rule as [monitoring-and-maintaining.md](monitoring-and-maintaining.md)). Steps beyond staring at the queue: check agent **config** if the instance was recently **cloned** (cloned envs carry the source's transport URIs — the classic "stage replicates to prod" incident), read the per-agent log directly at **`/etc/replication/agents.author/<AgentName>.log.html#end`**, then `error.log`, and if the queue is *idle* while content is missing, the problem is upstream — check the **workflow dashboard** (DAM Update Asset / launcher backlog), not replication. AEMaaCS equivalent failure is distribution queues instead: [content-distribution-journal.md](../cloud-service/content-distribution-journal.md).

## 6. TarMK corruption

Signatures in `error.log`: **`SegmentNotFoundException`**, `IllegalArgumentException`, "Error occurred while obtaining InputStream for blobId"; debug logger `org.apache.jackrabbit.oak.plugins.segment.file.TarReader-GC` (pre-6.3 package name — 6.3+ is `oak.segment`). Note [revision-cleanup.md](revision-cleanup.md)'s caveat first: an SNFE within 24 h of a revision's birth is usually app code holding a stale reference, not corruption.

Recovery (full backup first): **oak-run `check`** mode → "determine the last known good revision of the segment store" → manually revert the segment store to that revision (journal.log manipulation); if check reports **"No good revisions found"**, escalate to the article's part-B recovery. Afterwards: latest Service Pack + CFP + Oak CFP — corruption bugs get fixed in patch trains, staying unpatched re-rolls the dice. oak-run version must match Oak ([revision-cleanup.md](revision-cleanup.md)).

## Why this article earns a note

It's the closest thing to Adobe Support's internal decision tree: six symptoms → which console/MBean/log answers each. The durable part is the *mapping* (symptom → first artifact to pull), not the thresholds — those age.

Source: Adobe KCS ka-17457, fetched Oct 2026. 6.2-era content flagged where it conflicts with 6.5-current notes.
