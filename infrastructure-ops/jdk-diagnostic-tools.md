# JDK diagnostic tools (jstack, jmap, jcmd & friends) against a running AEM

Command-line tools that ship in the JDK's `bin/` next to `java` itself. They attach to a **running JVM by PID** and extract internal state without restarting it. Everything here applies to AEM 6.5 / AMS (you have a shell on the server). On AEMaaCS there is **no shell and no JVM access** — see the last section.

Two prerequisites that cause most "tool doesn't work" complaints:

1. **Same OS user.** The attach mechanism is a socket at `/tmp/.java_pid<pid>` owned by the process user. Run the tool as the user AEM runs as (`sudo -u aem jstack <pid>`), or you get `Unable to open socket file` / `Operation not permitted`. Root is *not* automatically enough.
2. **A JDK, not a JRE**, and ideally the **same JDK version** the target JVM runs on. Version-mismatched `jmap` fails with cryptic attach errors.

## Finding the PID: jps

```
jps -lv
```

`-l` prints the full main class / jar, `-v` the JVM args. An AEM quickstart shows up with its jar or `com.adobe.granite...` launcher plus the telltale `-Dsling.run.modes=...` args — which also lets you confirm *which* instance you're attached to when author and publish share a host. Same-user rule applies: `jps` run as the wrong user shows an empty list, not an error. Plain `ps -ef | grep java` works too and is what you want anyway before an upgrade ([in-place-upgrade-6x-to-65.md](in-place-upgrade-6x-to-65.md)).

## Threads: jstack — the CPU/hang/deadlock tool

```
jstack <pid> > threads-$(date +%H%M%S).txt
```

Prints every live thread with name, state (`RUNNABLE`, `WAITING`, `BLOCKED`...) and full stack trace, plus an automatic deadlock section ("Found one Java-level deadlock"). Output is plain text, hundreds of KB; the JVM pauses for milliseconds — effectively free, safe on production.

- `-l` adds ownable-synchronizer info (`java.util.concurrent` locks), worth having when hunting contention.
- **Always take a series**: 3–5 dumps, ~10 s apart. One dump is a photo; the series tells you whether the same thread shows the same top frame every time (genuinely stuck / hot loop) or was merely sampled mid-work. Interpretation gotchas (`RUNNABLE` ≠ working, idle pool workers that look blocked, AEM signature stacks) are in [thread-dump-analysis.md](thread-dump-analysis.md).
- **The high-CPU recipe** — correlate OS thread to Java stack:
  ```
  top -H -p <pid>          # per-thread CPU; note the hot LWP/thread id
  printf '%x\n' <lwp>      # convert decimal id to hex
  grep -A20 "nid=0x<hex>" threads-*.txt
  ```
  The `nid=` in the dump is the OS thread id in hex. This mapping is exactly what AEM's own auto-collected dumps **break** by default (`enableJStack=false` → every thread reads `nid=0xffffffff`; see [thread-dump-analysis.md](thread-dump-analysis.md)) — which is why you take your own `jstack` on a CPU case.
- `jstack -F` (force) for a JVM that won't respond to a normal attach: it uses the Serviceability Agent, halts the process while reading, gives less detail, and has been moved to `jhsdb jstack --pid <pid>` in newer JDKs. Last resort only.

## Heap: jmap — the memory/leak/OOM tool

```
jmap -dump:live,format=b,file=/big/disk/heap.hprof <pid>
```

Writes the entire heap — every object, its class, sizes, and the reference chains — as a binary `.hprof`. You then analyze offline in **Eclipse MAT** (dominator tree → "who is keeping this alive", Leak Suspects report) or VisualVM.

Costs, because they decide *when* you may run it:

- **Stop-the-world for the whole dump** — seconds to minutes on a production-sized AEM heap. On a box already struggling this makes things worse; on a publish farm, pull the node from the LB first.
- **File ≈ heap size** (multi-GB). Point `file=` at a disk that has the room — not the repository volume, which revision cleanup already wants 25% free ([revision-cleanup.md](revision-cleanup.md)).
- The `live` option runs a **full GC first** and dumps only reachable objects — smaller file, and garbage is excluded, which is usually what you want for leak hunting. Omit `live` only when you suspect premature collection / finalizer issues.

Cheap first look before committing to a full dump:

```
jmap -histo:live <pid> | head -30
```

Class histogram — instance counts and shallow bytes per class, no file. Two histograms a few minutes apart showing one class growing monotonically is often enough to name the leak without ever opening MAT.

For actual `OutOfMemoryError` cases, don't chase the process with jmap at 3 a.m. — have the JVM dump itself at the moment of death:

```
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/big/disk/dumps
```

These flags belong in the AEM start script of any production instance; the dump then captures the exact failing state, which a later manual dump never does.

## jcmd — the modern umbrella

One tool that supersedes most of the above on current JDKs, same PID + same-user mechanics:

```
jcmd <pid> help                      # list what this JVM supports
jcmd <pid> Thread.print -l           # = jstack -l
jcmd <pid> GC.heap_dump /path/h.hprof   # = jmap -dump (add -all for non-live)
jcmd <pid> GC.class_histogram        # = jmap -histo
jcmd <pid> GC.heap_info              # heap sizes/usage, instant
jcmd <pid> VM.flags / VM.system_properties / VM.command_line
jcmd <pid> JFR.start duration=120s filename=rec.jfr   # low-overhead flight recording
```

`jstack`/`jmap` survive mostly out of habit and old runbooks; `jcmd` is the one Adobe/Oracle keep investing in, and JFR (Java Flight Recorder, free since JDK 11) is the right answer when "which code burns CPU" needs *profiling over time* rather than snapshot dumps — near-zero overhead, open the recording in JDK Mission Control.

## GC behaviour over time: jstat

```
jstat -gcutil <pid> 1s
```

One line per second: occupancy of each generation + GC counts/time. The signature worth memorizing: **old gen pinned near 100% with full-GC count climbing and the app barely progressing = GC death spiral** — the JVM is burning CPU collecting instead of working. This *looks* like a CPU problem in `top` but is a memory problem; `jstat` is the 10-second test that routes you to jmap/MAT instead of jstack.

## Symptom → tool

| Symptom | First artifact |
|---|---|
| High CPU, suspected hot loop | `jstack` series + `top -H` correlation |
| Hang / requests stuck / deadlock | `jstack` (deadlock section, `BLOCKED` chains) |
| `OutOfMemoryError`, heap growth, leak | `jmap -dump` (or `HeapDumpOnOutOfMemoryError`) → MAT |
| "Is it a leak or just load?" quick check | `jmap -histo:live` twice, compare |
| High CPU that is actually GC | `jstat -gcutil` first, then heap dump |
| Need profiling over minutes, prod-safe | `jcmd JFR.start` → Mission Control |
| What flags/props is this JVM really running? | `jcmd VM.flags` / `VM.command_line` |

Cert-exam framing (AEM DevOps): a Support case about **high CPU** must include **thread dumps (jstack)**; bundle lists, heap dumps, and Sling metrics are the distractors — heap dumps are for the *memory* symptom family, and taking one on a CPU-pegged box actively hurts.

## AEMaaCS: none of these tools, same artifacts

No SSH, no pod shell, ephemeral instances — you cannot run jstack/jmap/jcmd yourself. Equivalents:

- **Thread dumps**: auto-collected on a 60 s schedule on cloud instances too ([thread-dump-analysis.md](thread-dump-analysis.md)), and obtainable via the **Developer Console**'s status/diagnostics for an environment.
- **Heap dumps**: through the Developer Console / Adobe Support rather than any self-serve CLI. (Exact self-serve availability per tier not screen-verified — treat "jmap on AEMaaCS" as always-wrong and "via Developer Console or Support" as the safe answer.)
- **Metrics/GC view**: Cloud Manager CPU/memory alerts and the provided APM tooling instead of `jstat`.

The rule of thumb from [aemaacs-repository-inspection-by-tier.md](../cloud-service/aemaacs-repository-inspection-by-tier.md) generalizes: any runbook step that assumes server access has a managed-surface replacement on AEMaaCS, and naming the on-prem tool is the exam trap.

Sources: JDK tool docs + Adobe troubleshooting KBs, Oct 2026; tool mechanics not re-verified against a live AEM for this doc except where linked docs say otherwise.
