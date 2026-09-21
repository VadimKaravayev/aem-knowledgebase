# TarMK Cold Standby (AEM 6.5 / AMS)

Warm-spare high availability for the **Author** tier on TarMK. On-prem and AMS only; AEMaaCS has no
equivalent and needs none (the platform manages author availability).

---

## What it is, and what it is not

One **primary** instance streams Oak segments one way to one or more **standby** instances. A standby
runs in sync-only mode: it serves no user requests and exposes only the OSGi Web Console for
administration. Failover is **manual**.

It is availability, not backup, and not scaling:

- **No integrity checking.** Segments replicate linearly, with no file or repository corruption
  detection. A corrupt primary produces a faithfully corrupt standby. Keep real backups.
- **Not for Publish.** Adobe explicitly rules this out; a publish farm is the answer there.
- **Not a capacity play.** A standby answers nothing. For author *capacity* see
  [multi-author-scaling.md](multi-author-scaling.md), which is a NodeStore decision (TarMK partitions,
  MongoMK clusters). Cold standby is what you reach for when the requirement is fail-safeness rather
  than throughput, and TarMK has to stay.

## How the sync works

The primary opens a TCP port (default **8023**) and listens. Each standby polls on `interval`
(default 5 seconds) with two message types: "what is the current head segment id" and "send me
segment X". Segments already held locally are skipped, referenced segments are pulled recursively.
Every packet carries a checksum, and a dropped connection retries automatically.

Cost profile: negligible on the primary (low CPU, no meaningful disk or network pressure). On the
standby, sync is **single-threaded and CPU-heavy**, idle when nothing changes. Sync duration tracks
change volume, hardware and network, *not* repository size, and SSL does not measurably slow it. The
first sync moves the whole repository and is network-heavy.

## Configuration

Configs live in a run-mode-specific install folder, `crx-quickstart/install.primary` and
`crx-quickstart/install.standby`, and the instance is started with the matching run mode:

```bash
java -jar quickstart.jar -r primary,crx3,crx3tar
java -jar quickstart.jar -r standby,crx3,crx3tar
```

**`org.apache.jackrabbit.oak.segment.standby.store.StandbyStoreService.config`**

| Property | Primary | Standby |
|---|---|---|
| `mode` | `"primary"` | `"standby"` |
| `port` | `I"8023"` | `I"8023"` |
| `primary.host` | n/a | `"127.0.0.1"` |
| `secure` | `B"false"` | must match primary |
| `interval` | n/a | `I"5"` (seconds) |
| `standby.autoclean` | n/a | `B"true"` |
| `standby.readtimeout` | n/a | 60000 ms default |
| `primary.allowed-client-ip-ranges` | IP allow-list | n/a |

**`org.apache.jackrabbit.oak.segment.SegmentNodeStoreService.config`**

Primary: `customBlobStore=B"true"`, `standby=B"false"`.
Standby: `name="Oak-Tar"`, `service.ranking=I"100"`, `standby=B"true"`, `customBlobStore=B"true"`.

An external data store needs its own `crx-quickstart/install/crx3` folder with its own config, e.g.
FileDataStore with `path="./crx-quickstart/repository/datastore"` and `minRecordLength=I"16384"`.

### Two configuration traps

1. **Set `org.apache.sling.installer.configuration.persist=B"false"` on production.** With
   persistence on, configuration lives in the repository, so the standby syncs the *primary's*
   configuration down on top of its own and stops being a standby.
2. **PIDs moved in AEM 6.3.** `org.apache.jackrabbit.oak.plugins.segment.*` became
   `org.apache.jackrabbit.oak.segment.*` for both `StandbyStoreService` and `SegmentNodeStoreService`.
   Configs copied from older notes are silently ignored.

Standard setup is: install and configure the primary, shut it down, copy the whole installation
folder to the standby host, then edit the standby's configs. Delete the standby's `sling.id` before
first start so the two instances do not share a repository id.

## Security

Enable `secure` on **both** sides (the setting must match) so segments are not streamed in the clear.
Restrict `primary.allowed-client-ip-ranges` so an arbitrary host cannot pull a full copy of the
repository. And make sure the load balancer can never route a user to a standby.

## Failover, manually

1. Stop the standby.
2. Remove the primary from the load balancer.
3. Back up the standby's `crx-quickstart`.
4. Restart it with the `primary` run mode.
5. Add it to the load balancer.
6. Build a fresh standby against the new primary.

## Monitoring (JMX)

`org.apache.jackrabbit.oak:type="Standby"`, plus the same bean on the primary keyed by port number.

Standby attributes: `Running`, `Mode` (client UUID, changes when config updates), `Status`
(`running` / `stopped`), `FailedRequests` (consecutive errors), `SecondsSinceLastSuccess`
(`-1` if never). **`SecondsSinceLastSuccess` is the staleness alarm**; `Running` only says the loop
is alive.

Primary attributes: `Mode` is always `primary`, plus per-client rows for up to 10 connections with
`Name`, `LastSeenTimestamp`, `LastRequest`, `RemoteAddress`, `RemotePort`, `TransferredSegments`,
`TransferredSegmentBytes`.

Both expose `start()`, `stop()` and `cleanup()`.

## Maintenance: order matters

**Online Revision Cleanup** (enabled by default on 6.3+) needs no manual procedure, and splits the
work across the pair: the **primary runs all three phases** (estimation, compaction, cleanup), and
the **standby automatically runs the cleanup phase only**, once the primary has finished. Estimation
and compaction never run on a standby, which is why compacting one by hand accomplishes nothing.
Verify it is working by checking that the standby's segmentstore size stays close to the primary's,
and watch the same `SegmentRevisionGarbageCollection` MBean and `TarMK GC` log lines as on the
primary. See [revision-cleanup.md](revision-cleanup.md).

**Offline revision cleanup**, if you are on it for some reason: stop sync via JMX, stop the primary,
compact the primary, start it, resume sync, wait for the logs to show sync complete, then invoke
`cleanup()` on the standby. Running offline revision cleanup *on* the standby is neither needed nor
effective.

**Data store GC**: finish the repository maintenance above first, then run GC on the primary and
afterwards on the standby. The standby has no `RepositoryManagement` MBean, so use
`BlobGarbageCollection#startBlobGC()` there. Order only matters for non-shared data stores.

**Hotfixes**: never patch a standby in place. Stop and delete it, install and test the hotfix on the
primary, stop the primary, file-system copy it, and reconfigure the copy as the standby.

## References

- [Deploying AEM / TarMK Cold Standby (Adobe docs, AEM 6.5)](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/implementing/deploying/deploying/tarmk-cold-standby)
- [multi-author-scaling.md](multi-author-scaling.md) for the capacity-vs-availability distinction.
