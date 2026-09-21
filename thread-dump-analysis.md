# Thread dump analysis in AEM (and why AEM's own dumps lie to you)

AEM collects thread dumps automatically, on by default, on both 6.5 and AEMaaCS. Two of its defaults make the output actively misleading if you read it like a `jstack` dump.

## Where they are, and what the file actually is

```
crx-quickstart/threaddumps/<YYYYMMDD>/threaddump.<HHMMSS>.txt.gz
crx-quickstart/threaddumps/<YYYYMMDD>.zip          # daily dirs get zipped
```

**Each file is 10 dumps, not one** — concatenated, 60 seconds apart, each with its own `<timestamp>` + `Full thread dump` header. The filename's `HHMMSS` is the rotation time, one period *after* the last dump inside. Split on the timestamp lines before analysing, or every count you make will be 10× inflated and every singleton thread (`Signal Dispatcher`, `main`) will appear ten times.

This is a gift: the "take a series of dumps several seconds apart" best practice is already done for you.

## The collector config

`com.adobe.granite.threaddump.impl.ThreadDumpCollector` — "Adobe Granite Thread Dumps Collector". Verified identical defaults on AEM 6.5 on-prem (`com.adobe.granite.threaddump-1.0.36`) and the AEMaaCS SDK:

| Property | Default | Notes |
|---|---|---|
| `granite.threaddump.enabled` | `true` | **on out of the box** |
| `scheduler.period` | `60` | seconds between dumps; missing value = collector never runs |
| `granite.threaddump.dumpsPerFile` | `10` | → 10-minute files at the default period |
| `granite.threaddump.enableJStack` | **`false`** | "Use native JStack JDK application to perform the thread dump" |
| `granite.threaddump.maxBackupDays` | `7` | **grab files before they rotate** |
| `granite.threaddump.enableGzipCompression` | `true` | per-file `.gz` |
| `granite.threaddump.enableDirectoriesCompression` | `true` | daily dir → `.zip` |
| `scheduler.runOn` | `SINGLE` | `SINGLE` = leader only; set "Each node" for all cluster nodes |
| `granite.threaddump.backupCleanTrigger` | `0 0 0 * * ?` | Quartz expression |

## Gotcha 1: `enableJStack=false` means no usable `nid`

With the default, dumps are generated from **ThreadMXBean**, not `jstack`/SIGQUIT. Tells:

```
"sling-default-1" prio=5 tid=0x89 nid=0xffffffff in Object.wait()
```

- `nid=0xffffffff` on **every** thread — a placeholder, not a native thread id.
- `tid` is a small sequential integer (`0x1`, `0x89`, …), i.e. `Thread.getId()`, not a native pointer.

**Consequence:** you cannot correlate a thread with `top -H -p <pid>` to find what is burning CPU. That correlation is the standard technique for CPU-pegged instances, and these files simply cannot support it.

**Fix:** set `granite.threaddump.enableJStack=true` (needs a JDK, not a JRE) to get real `jstack` output with real `nid`s. Or take your own: `jstack <pid>` / `jcmd <pid> Thread.print`, 5 dumps ~5–10s apart.

## Gotcha 2: "waiting to lock ... owned by null" is usually an *idle* thread

ThreadMXBean rendering produces this for parked threads, in bulk. A real observed file had **1,929** `waiting to lock` lines and **zero** `BLOCKED` threads. One of them:

```
"sling-default-1" prio=5 tid=0x89 nid=0xffffffff in Object.wait()
   java.lang.Thread.State: WAITING (on object monitor)
	at jdk.internal.misc.Unsafe.park(Native Method)
	- waiting to lock <0x1b6886f6> (a …AbstractQueuedSynchronizer$ConditionObject) owned by "null" tid=0x-1
	at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:435)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1070)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1130)
```

`ThreadPoolExecutor.getTask()` → `LinkedBlockingQueue.take()` is **an idle pool worker waiting for work** — the idlest state a thread has. `owned by "null" tid=0x-1` just means `ThreadInfo.getLockOwnerId()` returned `-1` (no owner).

**Never grep for `waiting to lock` to measure contention.** Judge by `Thread.State` and by whether application frames appear in the stack.

## Analysis order

1. **Split the file into its 10 dumps.** Count threads per dump: flat = steady state; monotonic climb that never falls = leak or a pool that can't drain; cliff = something died.
2. **State distribution per dump.**
3. **Deadlock section** — `Found one Java-level deadlock:`. The JVM does this for you; free answer when present.
4. **Diff across dumps.** Match threads by name; a thread with an *identical top frame in all 10* is genuinely stuck. Different frames = busy, not stuck. This is the entire payoff of a series.
5. **Follow the lock chain.** For each `BLOCKED` thread read `- waiting to lock <0xABC> owned by "thread-X"`, then go read `thread-X` — *it* is the bottleneck; the blocked ones are symptoms. Rank contention points by how many threads share the same `<0x…>`.
6. **Group by thread-name prefix** to identify the subsystem, then read the signature table below.

### Thread states

| State | Means | Worry? |
|---|---|---|
| `BLOCKED` | waiting on a monitor **another thread holds** | **Yes** — real contention |
| `RUNNABLE` | on CPU **or inside a native call** | Depends — see below |
| `WAITING` / `TIMED_WAITING` | parked | Usually idle workers |

**`RUNNABLE` does not mean working.** A thread in `socketRead0` / `SocketInputStream.read` is `RUNNABLE` while doing nothing but waiting on a remote server that may never answer. An outbound HTTP call with no socket timeout is one of the most common real AEM incidents and is **invisible** to anyone who only counts `BLOCKED` threads.

**Ignore idle pool workers.** No application frames above `ThreadPoolExecutor.getTask()` = noise. Frames from your own package = signal.

### AEM stack signatures

| Stack contains | Meaning |
|---|---|
| `SegmentNodeStore.merge`, `Commit` | Repository **write contention** — threads queued to commit. Classic AEM bottleneck. |
| `oak.plugins.index.AsyncIndexUpdate` | Async indexing lag |
| `oak-lucene`, `LuceneIndexEditor` | Index write load |
| `QueryEngineImpl`, `TraversingIndex` | **Unindexed query** traversing — cross-check `error.log` for `Traversed N nodes` |
| `socketRead0`, `PoolingHttpClientConnectionManager.leaseConnection` | External call hanging, or HTTP connection pool exhausted |
| all `qtp*` busy | Jetty pool exhausted — requests queue before AEM starts work |
| `sling-oak-observation-*` backing up | Observation listener too slow; event queue growing |
| `DamEventListener`, asset workflow threads | Ingestion storm |
| `ReplicationAgent`, `AgentManagerImpl` | Replication queue blocked |
| `synchronized` in your own package | Your code serializing |

### Correlate outward

A dump has no timeline. Pair with `request.log` (`crx-quickstart/opt/helpers/rlog.jar` sorts by slowest), `error.log` (exceptions, `Traversed N nodes`), the **GC log** (long pauses mimic hangs exactly), and `top -H` (only with `enableJStack=true`).

### Two artifacts that are always present and always meaningless

- `sling-default-N-com.adobe.granite.threaddump.impl.ThreadDumpCollector` in `ThreadImpl.dumpThreads0` — the observer effect: the thread taking the dump, in its own dump.
- Jetty selectors in `sun.nio.ch.KQueue.poll` / `EPoll.wait` and the acceptor in `Net.accept`, permanently `RUNNABLE`. Normal idle.

## Worked baseline: what healthy looks like

A real 10-dump file from an instance captured during startup:

```
                 24:37 25:37 26:37 27:37 28:37 29:37 30:37 31:37 32:37 33:37
threads             76   192   187   190   197   198   199   199   199   200
RUNNABLE            12    11    11    11    11    11    11    11    11    11
```

`main` still in `Felix.start()`/`Sling.<init>` in dump 1, settling to ~200 threads. Zero `BLOCKED`, no deadlock, `RUNNABLE` pinned at 11 and all of it infrastructure. The Sling job pool grew `0→14` monotonically but every worker was `WAITING` with no application frames — lazy pool expansion plus keep-alive lingering, **not** a leak. Growth in a pool only matters when the new threads carry application frames.

## Tools

- **fastthread.io** — paste, get grouping and deadlock detection. Fastest first pass; check data-sharing rules for customer dumps.
- **IBM TMDA** / **TDA** — offline alternatives.
- **`jstack <pid>`**, **`jcmd <pid> Thread.print`** — take your own when the automated ones are ThreadMXBean-flavoured.
