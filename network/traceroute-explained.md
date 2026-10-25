# traceroute: how it works, how to read it, and what it can't tell you about AEM

Run every example below on macOS. The traces were captured on 2026-10-24 from a home connection in Lviv to `cdn.adobeaemcloud.com`, which resolves to `adobe-aem.map.fastly.net`, Fastly `151.101.{3,67,131,195}.10`. Background on IP addresses and DNS: [ip-addresses-explained.md](../fundamentals/ip-addresses-explained.md).

## How it works

Every IP packet has a **TTL** (time to live): a hop counter that each router decrements. When it reaches 0, the router drops the packet and sends back an ICMP **"time exceeded"** message, which carries the router's own address.

traceroute uses this deliberately. It sends probes with TTL 1, 2, 3, … and each router on the path gives itself away when its probe expires. The destination is reached when the reply changes from "time exceeded" to a final answer: "port unreachable" for UDP, "echo reply" for ICMP, or SYN-ACK/RST for TCP.

Each line of output is one TTL. By default it sends three probes per TTL, so you get three round-trip times.

```
 1  192.168.0.1  46.419 ms  2.657 ms  2.680 ms      ← home router
 2  91.200.113.25  3.946 ms  4.010 ms  3.938 ms     ← ISP
 3  185.0.42.8  23.512 ms  22.999 ms  23.310 ms     ← internet exchange (fastly.1-ix.net)
 4  * * *                                           ← router that doesn't answer
 5  151.101.3.10  69.078 ms  23.524 ms  23.358 ms   ← Fastly, destination reached
```

## Probe protocol is the first thing to get right

| Tool / flag | Probe | Notes |
|---|---|---|
| macOS / Linux `traceroute` (default) | UDP to high ports (33434+) | Often dropped by firewalls and CDNs at the last hops |
| `traceroute -I` (macOS and Linux) | ICMP echo | What Windows `tracert` always uses |
| macOS `traceroute -P tcp -p 443` / Linux `traceroute -T -p 443` | TCP SYN | Closest to real HTTPS traffic. **Needs root:** without sudo, macOS fails with `pcap_activate() failed` |

**Verified gotcha:** with the default UDP mode, the trace to Fastly showed hops 1–3, then `*` all the way to the hop limit (`-m 18`). It never reached the destination. ICMP mode (`-I`) reached `151.101.3.10` at hop 5 in about 23 ms.

The site was fine. Fastly simply doesn't answer UDP probes. **Never conclude "unreachable" from a UDP-mode trace that ends in stars.** Re-run it with `-I`, then with TCP on port 443.

## Reading the output

- **`* * *` on a middle hop, with later hops answering:** that router just doesn't send "time exceeded" (or rate-limits it). It is not packet loss. Hop 4 above showed this on every run.
- **Stars from some hop all the way to the end:** either a filter is dropping your probe type (try another protocol) or the path really is broken. You can tell which only after trying `-I` and TCP.
- **A high time on a middle hop, but normal times after it:** ignore it. Routers answer traceroute on a slow, low-priority CPU path. Only latency that **persists through to the destination** is real.
- **Latency step that stays:** the jump from about 4 ms to about 23 ms between hops 2 and 3 above is real distance or queueing. It holds at every later hop.
- **The first probe is slow** (46 ms vs 2.6 ms at hop 1): ARP/cache warm-up. Ignore it.
- **The same hop number shows different IPs:** load balancing across parallel paths (ECMP). Normal.
- **Missing hops:** MPLS tunnels can hide whole stretches of a provider's network. "Five hops" doesn't mean five routers.
- **The path you see is one-way only.** The return path is often different (asymmetric routing), and trace timings include the return trip. A problem on the reverse path shows up in a forward trace as latency you can't explain.

## Useful flags (macOS)

```bash
traceroute -n host                      # no reverse DNS: faster, no stalls on hops with broken PTR records
traceroute -I host                      # ICMP probes
sudo traceroute -P tcp -p 443 host      # TCP SYN to 443 — matches HTTPS
traceroute -a -I host                   # prefix each hop with its AS number (network owner)
traceroute -q 1 -w 1 -m 20 host         # 1 probe/hop, 1 s wait, max 20 hops — quick
traceroute6 host                        # IPv6 (separate binary on macOS)
```

`-a` tells you **who owns each hop**. That shows where your ISP hands off and whether you're on the CDN's network yet:

```
 2  [AS43864] mz-gw.network.lviv.ua (91.200.113.25)   ← ISP
 3  [AS0] fastly.1-ix.net (185.0.42.8)                ← exchange point; reverse DNS names Fastly
 5  [AS54113] 151.101.3.10                            ← Fastly
```

`AS0` means the lookup didn't find an owner. Private IPs and exchange-point addresses typically show this way, so use the hostname instead.

## mtr: traceroute plus ping, for intermittent problems

A single trace is a snapshot. For "it's sometimes slow" use `mtr`, which traces repeatedly and shows per-hop loss and jitter. It isn't installed by default on macOS: `brew install mtr`, then run it with sudo.

```bash
sudo mtr -rwzc 100 -T -P 443 host   # report mode, wide, AS numbers, 100 cycles, TCP/443
```

The same rule applies: loss at a middle hop that **doesn't continue to the last hop** is rate-limiting, not loss. Only loss that persists to the destination counts.

## What this means for AEM

- **On AEMaaCS, a trace ends at the Fastly edge, not at AEM.** The `151.101.x.10` addresses are **anycast**: the same IPs are announced from every Fastly POP, so your trace stops at the POP nearest *you*. It says nothing about the CDN → origin (publish/dispatcher) path. You can't trace that path, and you have no shell on AEMaaCS to trace outward from it.
- **What a trace *can* answer for AEMaaCS:**
  - Does the customer domain resolve to and reach Adobe's CDN? Run `dig +short www.example.com` first: it should show the CNAME to `cdn.adobeaemcloud.com`, or the four Fastly A records for an apex domain ([cloud-manager-custom-domain-names.md](../cloud-service/cloud-manager-custom-domain-names.md)).
  - Is a slow user's problem on the way to the edge (their ISP or peering) or after it? If the trace reaches Fastly fast but pages are slow, it's caching or origin, not the network.
- **An IP allow-list block isn't a network problem.** The Cloud Manager allow lists ([cloud-manager-environments.md](../cloud-service/cloud-manager-environments.md)) are enforced *at the CDN*. A blocked office still traces cleanly to Fastly and then gets a 403 over HTTP. Use `curl -sv`, not traceroute, to debug them.
- **AMS / on-prem / self-hosted:** traceroute is useful here because you have shell access on the dispatcher and AEM hosts. Trace **from** the dispatcher to the publish instance (`sudo traceroute -P tcp -p 4503`, or `-T` on Linux), or from AEM to a backend or translation vendor, to find which firewall or segment drops traffic. Use TCP on the real port: security groups and firewalls commonly allow only that port, so a UDP or ICMP trace will "fail" on a path that works.
