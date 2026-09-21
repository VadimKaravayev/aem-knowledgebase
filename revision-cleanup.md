# Revision Cleanup: the parts that look like failures (AEM 6.4 / 6.5)

Every repository update writes a new revision, so the TarMK segment store only ever grows. Revision
cleanup reclaims the dead ones. **Online Revision Cleanup (ORC)** has been the default since 6.3 and
is what runs unless somebody turned it off.

This note deliberately skips what Adobe already documents well (how to set the maintenance window,
the oak-run command lines) and keeps the behaviour that reads as a malfunction when it is not.

**Sourcing:** written from the AEM **6.4** deploying/revision-cleanup page, cross-checked against the
6.5 cold standby page. Nothing here was reproduced on a local instance, so treat the thresholds as
documented values rather than observed ones.

---

## Three phases, and only the last one frees disk

1. **Estimation.** Has the repository grown by more than **1 GB** since the last compaction? If not,
   the run stops here and logs `TarMK GC: Size delta is ... so skipping compaction for now`. A quiet
   instance therefore shows days with no cleanup at all, which is correct behaviour.
2. **Compaction.** **Full** rewrites the whole repository (Sundays by default), **tail** rewrites
   only segments added since the last run (weekdays). The split is the `full.gc.days` property on
   the `RevisionCleanupTask`. Listing every day makes it full-only and tail never runs.
3. **Cleanup.** Deletes the old segments. **Disk space is reclaimed here or not at all**, so a
   compaction that completed with no change in footprint is not a bug, the run just has not reached
   or finished this phase.

Only revisions at least **24 hours** old are ever eligible.

## The results that look wrong and are not

- **The first ORC run reclaims nothing.** ORC keeps two generations, and on a first run no generation
  is old enough to collect. The same applies to the first ORC run after an offline cleanup.
- **Switching from offline to online doubles the repository.** Offline keeps one generation, online
  keeps two. It stabilises back around the pre-switch size on subsequent runs.
- **A full compaction grows the repository before cleanup shrinks it**, so alternating tail and full
  produces a sawtooth. Adobe notes that alternating the two modes delays reclamation.
- Generation sizes vary run to run, so the settled size is not a constant.
- **A crash mid-run is safe.** Leftover garbage is swept by the next run; there is no corruption risk.

## Two hard aborts and one silent skip

| Condition | Threshold | Configurable |
|---|---|---|
| Free disk below a share of the repository footprint | **25%** | **No** |
| Free heap below `SegmentNodeStoreService#MEMORY_THRESHOLD` | **15%** | Yes |
| Growth since last compaction | **1 GB** (skips the run) | Documented as fixed |

Size the disk **two to three times** the estimated repository size, largely because of these.

Under heavy write concurrency ORC may need exclusive write access and enters **forceCompact**, taking
a write lock with a **1 minute default timeout**; if it does not finish it aborts in favour of the
concurrent commits rather than blocking them. Duration scales with segment store size and inversely
with IOPS. A healthy run is expected to finish inside **2 hours**.

## Monitoring

JMX MBean `org.apache.jackrabbit.oak:type="SegmentRevisionGarbageCollection"`, attribute
`EstimatedRevisionGCCompletion` for progress, `startRevisionGC()` to trigger a run by hand. Logs are
prefixed `TarMK GC #n:` per phase, with failures at WARN or ERROR under the same prefix.

The **Revision Cleanup Health Check** in the Operations Dashboard is green after a success, yellow
after one cancellation, and **red after three consecutive cancellations**, meaning manual
intervention. **Gotcha: the status resets on restart**, so a routine restart erases the only
accumulated signal that cleanup has been failing.

## Scheduling gotcha

Maintenance tasks run **sequentially with no configurable order**. Putting ORC in a window shared
with other maintenance tasks delays it behind them, so give it its own.

## `SegmentNotFoundException`

Three causes, in likelihood order: application code holding a segment reference **past the 24 hour
retention** (by far the common one, and a code bug rather than a cleanup bug), external disk
corruption, or something that needs Customer Care. Verify repository integrity with oak-run.

## If you do end up on offline cleanup

Reserved for migrations or when Adobe Customer Care asks. It keeps one generation and can therefore
reclaim more aggressively, because ORC has to respect revisions the running application stack may
still reference.

Version-match matters: **oak-run must match the Oak core version**. Two flags on the 6.4 page are
legacy and should not be copied forward: `-Dsun.arch.data.model=32` and
`-Dtar.PersistCompactionMap` (removed in Oak 1.6). `--force` upgrades the segment store format and
makes it unreadable by older Oak, so it is one-way. Clean up unreferenced checkpoints first
(`oak-run.jar checkpoints <segmentstore> rm-unreferenced`); more than three checkpoints present
during compaction is a signal to do so.

Note that migrating to the 6.4 persistence format is **not reversible** back to 6.3.

## Windows

Memory-mapped file access is never used on Windows, regular file access is always enforced. Compensate
by giving the heap everything you can spare and raising `segmentCache.size` in
`org.apache.jackrabbit.oak.segment.SegmentNodeStoreService.config` (example: `segmentCache.size=20480`),
leaving RAM for the OS. On other platforms, more physical memory improves mapping of the repository.

## Cold standby

The primary runs all three phases; the **standby automatically runs the cleanup phase only**, after
the primary completes. Estimation and compaction never run on a standby. See
[tarmk-cold-standby.md](tarmk-cold-standby.md).

## References

- [Revision Cleanup (Adobe docs, AEM 6.4)](https://experienceleague.adobe.com/en/docs/experience-manager-64/deploying/deploying/revision-cleanup)
- [tarmk-cold-standby.md](tarmk-cold-standby.md)
