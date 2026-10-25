# Analyze memory problems — Adobe Support's OOM recipe (KCS ka-17482 / KA-02700)

Adobe's own knowledge-base article for an AEM/CRX JVM that slows down, runs out of memory, or logs `OutOfMemoryError: Java heap space` / `OutOfMemoryError: GC overhead limit exceeded`. CQ-vintage (it still says "CRX/CQ" and quotes a 256 MB minimum heap) but the **method** is what Support expects you to have followed before opening a ticket. Applies to **6.5 / AMS / any self-hosted JVM**; on **AEMaaCS none of this is possible** — heap dumps come from Adobe via a Support request, you work from Developer Console logs instead ([[jdk-diagnostic-tools]], [[aemaacs-repository-inspection-by-tier]]).

The deeper tool reference (attach prerequisites, stop-the-world cost, `jcmd`, JFR) is [[jdk-diagnostic-tools]]; this page is the Support-flavoured checklist on top of it. Article read 2026-10-25, last updated by Adobe 2026-09-17.

## The recipe, in Adobe's order

1. **Check `-Xmx` first.** No explicit heap on the command line = JVM default, which is the most common "OOM" on a fresh box. The article's floor is `java -Xmx512m -jar *.jar` for CRX/CQ; real 6.5 instances run 8–16 GB+. If the heap is already right, assume **retained objects (a leak)** and continue.
2. **Let the JVM dump itself at the moment of death**: add `-XX:+HeapDumpOnOutOfMemoryError`. The `java_pid<pid>.hprof` lands in the process **working directory**, the process **keeps running** afterwards, and **one dump is usually enough**. Redirect with `-XX:HeapDumpPath=/big/disk/dumps` when the working directory is short on space.
   - Start-script instances: put it in `CQ_JVM_OPTS` in the start script (article says `crx-quickstart/server/start` — CQ 5; on 6.x it is `crx-quickstart/bin/start`) and make sure the line is **uncommented**.
   - **Verify after restart** with `ps -ef | grep java` — the flag must be visible on the CRX process. A commented-out `CQ_JVM_OPTS` line is the classic reason "the dump never appeared".
3. **Manual dump, HotSpot (Oracle/OpenJDK)**: PID via `ps -ef | grep java` or `jps -l` (Windows: Task Manager with the PID column, or `jps -l`), then from the **JDK** `bin`:
   ```
   jmap -dump:format=b,file=/path/to/heapdumpfile.hprof <pid>
   ```
4. **Manual dump, IBM JVM**: enable a user-signal system dump with
   ```
   -Xdumpopts=PHD+CLASSIC:events=user -Xdumpevents=user
   ```
   then trigger with **SIGQUIT** (Linux/AIX/z/OS/i5) or **SIGBREAK** (Windows). Adobe says a **full system dump** is generally needed for a thorough analysis on IBM.
5. **Disk warning**: a heap dump can be **as large as `-Xmx`**. Check free space on the target directory before step 2 or 3 — never the repository volume ([[revision-cleanup]] already wants 25 % free there).
6. **Analyse**: HotSpot `.hprof` → **Eclipse Memory Analyzer (MAT)**. IBM PHD/classic dumps → **IBM HeapAnalyzer**; IBM full system dumps → **IBM Support Assistant Workbench + Monitoring and Diagnostic Tools for Java – Memory Analyzer**. **MAT cannot read IBM dumps.**
7. **Optional cheap first look — heap histogram**: `jmap -histo <pid>` (live object count + bytes per class, sorted by shallow size). **The first ~20 lines are the useful part.** Tools may be missing or unreliable depending on the Java install.
8. **What to send Support**: CRX/CQ (AEM) version **including every installed hotfix/service-pack number**, OS, JVM vendor and JVM version — plus the dump.

## What the article does not say, and you need

- `jmap` must run as the **same OS user** as the AEM process, with a **version-matched JDK** — otherwise cryptic attach errors ([[jdk-diagnostic-tools]]).
- `jmap -dump` is **stop-the-world** for the whole write; on a publish node pull it from the load balancer first.
- Take `jmap -histo:live` **twice a few minutes apart** — the class whose count only grows is usually the leak, often without opening MAT.
- `jcmd <pid> GC.heap_dump` / `GC.class_histogram` supersede `jmap` on current JDKs; `JFR.start` is the production-safe way to watch allocation over time.
- `GC overhead limit exceeded` with a *correctly sized* heap is the leak signature; `Java heap space` right after start on a default heap is the sizing one.
- For "slow but no OOM", correlate with [[thread-dump-analysis]] first — a GC storm shows up as all threads parked.

## Exam / triage traps

- Raising `-Xmx` as the fix for a leak — buys time, moves the OOM, and makes the eventual dump bigger.
- Setting `HeapDumpOnOutOfMemoryError` and not checking `ps -ef` — commented-out `CQ_JVM_OPTS` means no dump.
- Writing the dump to the repository disk — a multi-GB file beside the segment store can tip revision cleanup or the whole instance over.
- Opening an IBM dump in MAT and concluding it is corrupt.
- Filing the Support ticket without hotfix versions — first reply is a request for them.

## References

- Adobe KCS *Analyze Memory Problems*: https://experienceleague.adobe.com/en/docs/experience-cloud-kcs/kbarticles/ka-17482
- Oracle HotSpot troubleshooting guide (memory leaks) and JVM debugging options, as cited by the article.
