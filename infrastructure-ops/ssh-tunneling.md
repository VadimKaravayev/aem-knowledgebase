# SSH tunneling — reaching firewalled AEM ports (debug, JMX, admin UIs)

An SSH tunnel is **port forwarding carried inside an SSH session**: you authenticate to a
server you *are* allowed to reach over SSH (22), and the SSH connection relays traffic for
some other port that the firewall does *not* expose. Nothing on the server changes — sshd
just proxies bytes. This is the standard way to reach an AEM instance's JVM debug port, JMX,
or even the author UI on a locked-down AMS/on-prem box where only SSH is open.

## The three modes

| Mode | Flag | Direction | Typical use |
|---|---|---|---|
| Local forward | `-L local:host:port` | your machine → server | reach a remote port as if it were local (the common case) |
| Remote forward | `-R remote:host:port` | server → your machine | expose your local service to the remote side (webhook callbacks, rare in AEM work) |
| Dynamic (SOCKS) | `-D port` | your machine → anywhere the server can reach | browser proxy into a private network — reach *all* of an environment's internal hosts/UIs through one tunnel |

The workhorse — bring a remote AEM debug port to your laptop:

```
ssh -L 30303:localhost:30303 user@aem-server
```

Then point the IDE's remote-debug config at `localhost:30303`. `localhost` in the middle of
the `-L` spec is resolved **on the server**, so `-L 4502:publish-internal:4502` through a
bastion reaches a host *behind* it that your laptop cannot resolve at all. Useful extras:
`-N` (forward only, no shell), `-f` (background), `-o ServerAliveInterval=60` (keep
long-lived tunnels from being dropped by idle timeouts).

## The layering point (exam-relevant)

A tunnel is pure **transport** — it forwards a port, it cannot make the JVM *produce*
anything on that port. Remote debugging requires the JDWP agent enabled at JVM startup:

```
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:30303
```

(or the quickstart `-debug` switch, which adds the same). Cert-exam framing (AEM DevOps):
"how do developers step through running code?" → **start AEM with the remote-debugging JVM
parameter**; "create an SSH tunnel to the java port" is the distractor — with no JDWP agent
there is nothing listening on the other end of the tunnel. Same split in real work: the
tunnel is step 2, the startup flag is step 1 and needs a restart.

JMX has the same shape plus its own trap: `com.sun.management.jmxremote.port` is only the
*registry* port — RMI then hands the client a second, by-default-random port, so a plain
single-port tunnel breaks. Fix by pinning `com.sun.management.jmxremote.rmi.port` to the
same value and tunneling that one port, and prefer `jmxremote.local.only=true` +
`-D` SOCKS or a tunnel over opening JMX through the firewall at all.

## Security posture

- A tunnel inherits SSH's auth and encryption — it is the *sanctioned* alternative to
  opening debug/JMX/4502 ports on a firewall. Debug and JMX ports themselves are
  unauthenticated (JDWP allows arbitrary code execution), so they must bind to
  `localhost`/private interfaces and be reached only by tunnel; `address=*:30303` on a
  public interface is an incident, not a convenience.
- Ops teams can disable forwarding server-side (`AllowTcpForwarding no`) — if `-L` silently
  connects nowhere, check that before debugging the client.

## AEMaaCS: does not exist

No SSH to cloud instances, no pod shell, ephemeral pods — tunneling is an AMS/on-prem
technique only. The managed replacements: Developer Console status/diagnostics for what
you'd have pulled over a tunnel ([jdk-diagnostic-tools.md](jdk-diagnostic-tools.md)), and
**local** debugging against the SDK quickstart instead of remote-debugging a cloud env.
Any answer that assumes SSH access to AEMaaCS is wrong by construction
([aem-critical-issue-triage.md](aem-critical-issue-triage.md) pattern: name the managed
surface, not the server tool).

---

Written Oct 2026 from working knowledge of OpenSSH and JDWP/JMX mechanics; flags
cross-checked against OpenSSH man-page semantics from memory, not re-verified against a
live system. The exam framing is from the DevOps question bank (question-2).
