# Thread dump capture methods (Adobe KCS ka-17452)

[KCS article](https://experienceleague.adobe.com/en/docs/experience-cloud-kcs/kbarticles/ka-17452) "Take thread dumps from a JVM" — Adobe Support's capture recipes, for all JVM products with AEM 6.x callouts. This note is the **capture** side; interpretation lives in [thread-dump-analysis.md](thread-dump-analysis.md) and tool context in [jdk-diagnostic-tools.md](jdk-diagnostic-tools.md).

## The standard recipe

1. PID via `jps -l` (as the AEM OS user: `sudo -u <user> jps -l`; fallback `ps -el | grep java`).
2. `jstack -l <pid> >> threaddumps.log` — **`-l` adds lock/synchronizer detail** (worth it; the plain form omits it), run **as the same OS user** as the process (root isn't enough — the attach-socket rule from [jdk-diagnostic-tools.md](jdk-diagnostic-tools.md)).
3. **Series, not snapshot**: Adobe's number here is "**10 dumps at a regular interval (for example, one every ten seconds)**" — more generous than the 3–5 × ~10 s habit; either way, one dump proves nothing, diffing dumps does.

`jstack` works even with `-Xrs` set; on a 64-bit JVM that won't respond, `jstack -J-d64 -l <pid>`.

## The two tricks worth remembering

- **`jstackSeries.sh`** — Adobe's own loop script at [github.com/cqsupport/jstackSeries.sh](https://github.com/cqsupport/jstackSeries.sh): `sudo -u <user> jstackSeries.sh <pid> <aemserveruser> <count> <delay>` — plus it captures `top` output alongside each dump, giving the thread-CPU correlation data in one shot.
- **Windows, jstack failing with an access/storage error**: AEM often runs as a service under SYSTEM, and jstack must match — `psexec -s jstack <pid> >> threaddumps.log` (psexec's `-s` = run as SYSTEM). The Windows twin of the same-user rule.

## Fallbacks

- `kill -QUIT <pid>` (SIGQUIT) — the JVM prints a dump to **its own stdout**, not your terminal; the article points at `crx-quickstart/server/logs/startup.log` (CQ5-era path — on modern 6.x look in `stdout.log` or wherever stdout is redirected, [monitoring-and-maintaining.md](monitoring-and-maintaining.md)). No JDK tools needed.
- **`/system/console/status-Threads`** — Felix console runtime view (6.x only, no console on AEMaaCS). **Caveat the article states: its format is incompatible with analyzers like TDA or Samurai** — same genus as the ThreadMXBean rendering quirks of AEM's auto-dumps ([thread-dump-analysis.md](thread-dump-analysis.md)). Fine for a quick look, wrong artifact to attach to a Support ticket.
- jconsole with the TDA plugin (`jconsole.exe -pluginpath tda.jar`) for JMX-based capture on old Windows JDKs.

Analyzer shortlist per Adobe: [TDA](https://github.com/irockel/tda), [fastthread.io](https://fastthread.io/) (upload — mind what's in your stack traces), IBM Thread Monitor and Dump Analyzer.

Source: Adobe KCS ka-17452, fetched Oct 2026. Capture commands not re-verified locally.
