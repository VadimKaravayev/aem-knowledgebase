# Remote-debugging AEM from the IDE (JDWP) — KCS ka-17459 plus what it leaves out

Adobe Support's recipe for stepping through code on a local or remote AEM instance: start the JVM with the **JDWP agent**, attach the IDE's "Remote Java Application" configuration. Article *How to debug an AEM app using Eclipse*, applies to **6.4 / 6.5** (AMS/on-prem); read Oct 2026, Adobe last update 2026-08-18. On **AEMaaCS there is no JDWP** — no JVM flags, no port; the fidelity tools are RDE + logs ([[local-development-environment]], [[aemaacs-repository-inspection-by-tier]]).

Transport to a firewalled debug port is a separate concern: [[ssh-tunneling]].

## Step 1 — enable the agent at JVM start

```
-agentlib:jdwp=transport=dt_socket,address=8000,server=y,suspend=n
```

Two ways in, per the article:

- **Permanent**: add it to `CQ_JVM_OPTS` in `crx-quickstart/bin/start` (uncomment the line) so the instance always starts debuggable.
- **One-off**: `java -Xmx512m -agentlib:jdwp=…,address=8000,… -jar cq-author-4502.jar`.

Notes from the article: the port (8000) is free to change; if the debugger will not attach, start with **`-nofork`** so the quickstart does not fork a child JVM that never received the flag; the start script lives in `crx-quickstart/bin`.

⚠ Adobe's warning: **do not leave remote debugging enabled in production** — it costs performance (JIT de-optimisation, slower exception paths, and an open unauthenticated port).

## Step 2 — attach from the IDE

Eclipse (article): Run → Debug Configurations → right-click **Remote Java Applications** → New → pick the AEM project → host (`localhost` or the tunnelled host) + the `address` port → save, double-click to start.

IntelliJ: Run → Edit Configurations → **+ Remote JVM Debug** → same host/port; it prints the exact `-agentlib:jdwp` string to copy. Breakpoints only bind when the IDE module's compiled classes match the deployed bundle — redeploy the bundle after every change, or the lines drift.

## What the article leaves out

- **`address=8000` on JDK 9+ binds to localhost only.** For a remote box use `address=*:8000` (or `address=0.0.0.0:8000`) — or keep it local and tunnel in, which is the safer default ([[ssh-tunneling]]).
- **Quickstart shortcut**: `java -jar aem-author-p4502.jar -debug 8000` (or the `-debug` switch in the start script) adds the same agent string; the start script's own `-debug` handling and `CQ_JVM_OPTS` must not both set it or the JVM refuses to start on a duplicate agent.
- `suspend=y` makes the JVM wait for a debugger before running a single line — the way to catch startup code (repoinit, activators, `SlingRepositoryInitializer`), otherwise never.
- **Hot swap** works for method-body edits only; adding a method, field or class needs a bundle redeploy (`mvn -PautoInstallBundle`).
- On a **publish farm / clustered author** attach to one node and pull it from the load balancer; a breakpoint holds a request thread and the health check will fail it anyway.
- **AMS**: ask for the flag to be set via a CSE ticket; the port is reachable only through the bastion.

## Exam / triage traps

- "Create an SSH tunnel to the Java port to debug" — a tunnel forwards a port; without `-agentlib:jdwp` nothing is listening ([[ssh-tunneling]]).
- Debugger attaches but no breakpoint ever hits → the flag went to the launcher, not the forked JVM: use `-nofork` or put it in `CQ_JVM_OPTS`.
- Attaching to a remote host fails with "connection refused" on a modern JDK → `address=8000` bound to loopback; use `*:8000` or tunnel.
- Leaving `suspend=y` in `CQ_JVM_OPTS` — the instance "hangs on start" until someone attaches.
- AEMaaCS questions offering JDWP as an option — always a distractor.

## References

- Adobe KCS *How to debug an AEM app using Eclipse*: https://experienceleague.adobe.com/en/docs/experience-cloud-kcs/kbarticles/ka-17459
