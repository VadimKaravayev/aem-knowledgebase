# ICMP: the internet's status and error channel, and what it means for AEM diagnostics

ICMP (Internet Control Message Protocol) does not carry application data. It carries messages *about* other packets: "that didn't arrive", "that was too big", "that ran out of hops", "yes, I'm here". Examples target `cdn.adobeaemcloud.com` (Fastly anycast, `151.101.{3,67,131,195}.10`). Companion docs: [traceroute.md](traceroute.md) (built entirely on one ICMP message type) and [ip-addresses.md](ip-addresses.md).

## The post-office picture

You mail a letter. If the address doesn't exist, the post office stamps it "return to sender: no such address" and sends it back. If a sorting centre loses it, nothing comes back at all. ICMP is the stamp: routers and hosts use it to tell the *original sender* why a packet could not be delivered, or to confirm reachability.

Two consequences follow from the analogy and hold in practice:

- **No stamp does not mean delivered.** ICMP errors are best-effort and widely filtered or rate-limited. Silence is ambiguous (see "Reading silence" below).
- **The stamp goes to the sender, not the recipient.** The destination host never learns a packet was dropped; the client does. This is why a client-side `curl -v` or `ping` sees a problem the server logs never will.

## Where it sits

ICMP rides directly on IP (protocol number 1; ICMPv6 is 58), alongside TCP and UDP. It is **not** a transport protocol:

- **No ports.** A firewall rule "allow ICMP" cannot be scoped by port; it is scoped by *type* and *code* (below).
- **No connection, no retransmission.** Lost ICMP is simply lost.
- **Applications rarely send it.** The OS kernel and routers generate it automatically. `ping` and `traceroute` are the exceptions that craft it deliberately, which is why they need raw sockets (root, or a setuid binary).

Every ICMP *error* message quotes the IP header plus the first 8 bytes of the packet that triggered it. Those 8 bytes contain the TCP/UDP source and destination ports, which is how the sending OS matches the error to the right socket. You can see the quoted header in macOS `ping` output:

```
$ ping -c 1 -m 2 cdn.adobeaemcloud.com          # -m 2: TTL 2, expires at the ISP router
36 bytes from gw.isp.example.net (203.0.113.1): Time to live exceeded
Vr HL TOS  Len   ID Flg  off TTL Pro  cks      Src          Dst
 4  5  00 5400 98e1   0 0000  01  01 84d6 192.0.2.10     151.101.67.10
```

The second line is the header of *your* original echo request as the router received it (`Pro 01` = ICMP, TTL already decremented to `01`). This is also why a NAT router can translate ICMP errors correctly: it rewrites the quoted inner header too.

## Header anatomy

The ICMP header is a fixed 4-byte prefix, 4 bytes whose meaning depends on the type, then a payload whose shape also depends on the type. It sits directly after the IP header, in the IP payload.

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-------------------------------+-------------------------------+
|     Type      |     Code      |           Checksum            |
+-------------------------------+-------------------------------+
|                 "Rest of header" (4 bytes, type-specific)     |
+---------------------------------------------------------------+
|                 Data (type-specific, variable)                |
```

**The fixed 4 bytes**

- **Type (1 byte).** What kind of message: 8 Echo Request, 0 Echo Reply, 3 Destination Unreachable, 11 Time Exceeded, 5 Redirect, 12 Parameter Problem.
- **Code (1 byte).** Sub-reason within the type. Type 3: 0 net, 1 host, 3 port, 4 fragmentation needed, 13 administratively prohibited. Type 11: 0 TTL expired in transit, 1 reassembly timeout. Echo is always code 0.
- **Checksum (2 bytes).** Ones-complement sum over the whole ICMP message, header and data. **No pseudo-header in IPv4**, so the IP addresses are not covered. A NAT that rewrites the quoted inner header must also fix this checksum. How the sum works: [internet-checksum.md](internet-checksum.md).

**The second 4 bytes, by type**

| Type | Bytes 4-5 | Bytes 6-7 |
|---|---|---|
| Echo 8 / 0 | Identifier (usually the process ID) | Sequence number (the `icmp_seq=` in ping output) |
| Destination Unreachable 3, code 4 | unused (zero) | **Next-hop MTU**: the value PMTUD needs |
| Destination Unreachable 3, other codes | unused | unused |
| Time Exceeded 11 | unused | unused |
| Redirect 5 | Gateway IP address (all 4 bytes) | |
| Parameter Problem 12 | Pointer (1 byte: offset of the bad field) | unused |

**The data section, by kind of message**

- **Echo.** Arbitrary payload, echoed back byte for byte. macOS `ping` sends 56 data bytes by default, so the ICMP message is 64 bytes: the `64 bytes from` in the output. The first 8 bytes are a send timestamp so ping can compute the round trip without keeping state. `-s` sets the size.
- **Errors (types 3, 11, 5, 12).** The IP header of the offending packet plus at least the first 8 bytes of its payload. Those 8 bytes hold the TCP or UDP source and destination ports, which is how the sending kernel finds the socket to deliver the error to. RFC 1812 lets routers quote more, up to 576 bytes total, and Linux does. This is the header `ping -m 2` printed above.

**Sizes to keep in your head**

| Message | ICMP bytes | On the wire with a 20-byte IPv4 header |
|---|---|---|
| Minimal error (8 header + 20 quoted IP header + 8 quoted payload) | 36 | 56 |
| Default macOS echo request (8 header + 56 data) | 64 | 84 |

The `36 bytes from` on the Time Exceeded line above is that minimal error size; the `92 bytes from` the LAN router printed is a router that quoted the whole 84-byte original packet (8 + 84).

**ICMPv6 differences.** Same 4-byte prefix layout, but the checksum **does** include an IPv6 pseudo-header with both addresses. Types are renumbered: Destination Unreachable 1, Packet Too Big 2 (MTU in the second 4 bytes), Time Exceeded 3, Parameter Problem 4, Echo 128/129. **Types below 128 are errors, 128 and above are informational**, so a firewall can express "allow all errors" with one range rule. Errors quote as much of the original packet as fits in the IPv6 minimum MTU of 1280 bytes.

**Seeing it yourself**

```bash
sudo tcpdump -i en0 -nvv -X icmp
```

The `-X` hex dump shows the type and code bytes at the start of the ICMP section, right after the 20-byte IP header. On an error a second IP header starts 8 bytes later, beginning with `45` (IPv4, 20-byte header).

## The message types you will actually meet

| Type | Name | Who sends it | What it tells you |
|---|---|---|---|
| 8 / 0 | Echo Request / Echo Reply | `ping` | Host is reachable at IP level and the round trip time. Nothing about ports, TLS, or the application. |
| 3 | Destination Unreachable | Router or destination host | Delivery failed. The **code** says why: 0 net, 1 host, **3 port unreachable** (UDP to a closed port; how UDP traceroute knows it arrived), **4 fragmentation needed and DF set** (Path MTU Discovery), 13 administratively prohibited (a firewall admitting it dropped you; most don't). |
| 11 | Time Exceeded | Router | TTL hit zero (code 0). This is the whole mechanism behind traceroute. Code 1 = reassembly timeout. |
| 5 | Redirect | Router | "Use this other gateway for that destination." Hosts on a LAN with two routers see it; most modern OSes ignore it for security. |
| 4 | Source Quench | nobody any more | Deprecated congestion signal. If a question mentions it, the answer is "obsolete, not sent". |

ICMPv6 keeps these (Echo 128/129, Destination Unreachable 1, Packet Too Big **2**, Time Exceeded 3) and adds **Neighbor Discovery** (types 133 to 137), which replaced ARP. Blocking ICMPv6 wholesale therefore breaks IPv6 entirely, not just diagnostics.

## Rules that prevent message storms

A router never sends an ICMP error:

- in response to another ICMP *error* (it will respond to an Echo Request, which is not an error);
- for a packet addressed to a broadcast or multicast address;
- for any fragment other than the first;
- for a packet whose source address is not a single host (zero, loopback, broadcast).

And hosts rate-limit the errors they do send (Linux default is one per second per type per destination via `icmp_ratelimit`). Expect gaps when probing fast.

## Reading silence

Because ICMP is optional and filtered, the absence of a reply has at least four readings. Verified cases:

| Observation | Natural reading | What it actually meant here |
|---|---|---|
| `ping host` gets no reply | host is down | **Not conclusive.** Many hosts and most cloud load balancers drop Echo Requests. Adobe's Fastly edge *does* answer (4/4 replies, ~23 ms), so for AEMaaCS silence is a real signal; for an on-prem Dispatcher behind a hardened firewall it usually is not. |
| UDP `traceroute` ends in `* * *` | path broken | Destination doesn't send Port Unreachable for UDP. `-I` (ICMP) reached the same host in 5 hops. See [traceroute.md](traceroute.md). |
| `nc -u -z host 33434` prints `succeeded!` | UDP port open | **UDP "open" only means no Port Unreachable came back within the timeout.** The router here silently dropped it; `nc` cannot tell the difference. Never trust UDP port scans. |
| `ping -D -s 1473 host` prints `sendto: Message too long` | the network rejected it | The **local kernel** refused before sending: 1473 + 8 ICMP + 20 IP = 1501 bytes > the interface MTU of 1500. No ICMP was involved. A *remote* MTU limit would instead show a Fragmentation Needed message, or (when it is filtered) nothing at all, which is the PMTUD black hole below. |

## The gotcha that bites production: Path MTU Discovery

Modern hosts send all TCP segments with the Don't-Fragment bit set (MTU values, MSS arithmetic and the probing recipe: [mtu.md](mtu.md)). When a link on the path has a smaller MTU (VPNs, PPPoE, GRE/IPsec tunnels, some cloud interconnects), the router drops the big packet and returns **Fragmentation Needed (type 3 code 4)** with the MTU it can carry. The sender shrinks its segment size and retries. Nobody notices.

If a firewall anywhere on the path drops type 3 code 4, that feedback never arrives:

1. TCP handshake succeeds (SYN packets are tiny).
2. The TLS ClientHello / first large response is dropped repeatedly.
3. The connection hangs until a timeout. Small pages work, large ones don't. `ping` works. `telnet host 443` connects.

That symptom pattern ("connects but hangs on anything big") is a **PMTUD black hole**, also called a **TCP black hole** or **MTU black hole**, until proven otherwise. ("Black hole" generally = packets dropped silently with no error; a firewall `DROP` is a black hole and fails slowly via timeout, a `REJECT` sends ICMP unreachable or TCP RST and fails instantly.) Fixes, in order of preference: allow ICMP type 3 code 4 (and ICMPv6 Packet Too Big) through the firewall; enable TCP MSS clamping on the tunnel/VPN device; as a last resort enable `net.ipv4.tcp_mtu_probing=1` on Linux senders so TCP finds the MTU without ICMP.

**Firewall rule of thumb:** if you must block ICMP, block Echo Request only. Always allow Destination Unreachable (at least code 4), Time Exceeded, and for IPv6 all of Neighbor Discovery plus Packet Too Big.

## What `ping` can and cannot tell you

- **Can:** the host's IP stack is up, the forward and return paths exist, and the round trip time at that moment. A rising `time=` with stable loss points at queueing; loss that persists to the destination in `mtr` is real.
- **Cannot:** whether port 443 is open, whether TLS handshakes, whether the application answers, or whether *your* traffic class is treated the same. Routers and CDNs prioritise ICMP differently from TCP; a clean ping with a slow site is normal, and so is a dropped ping with a fast site.
- **TTL in the reply** tells you how many hops the *return* path took: `ttl=60` from Fastly means 64 minus 4 hops back. Different TTLs on repeated pings to the same name mean anycast or ECMP is handing you different servers.

Useful macOS forms:

```bash
ping -c 4 host                 # four echo requests
ping -c 1 -m 3 host            # TTL 3: which router is third on the path (one-shot traceroute line)
ping -D -s 1400 host           # Don't-Fragment with 1400 data bytes; probe the path MTU step by step
ping -i 0.2 -c 50 host         # faster interval; sub-0.1 s needs root
ping6 host                     # ICMPv6 echo
```

## What this means for AEM

- **AEMaaCS:** `ping cdn.adobeaemcloud.com` reaches the nearest Fastly POP, not Adobe's origin. It answers Echo, so it is a fair "does my network reach Adobe's edge" check and nothing more. Cloud Manager IP allow lists are enforced at the CDN as HTTP 403, not as ICMP; a blocked office pings fine. CDN → origin problems are invisible to any ICMP tool you run; use `curl -sv` and the `X-Cache`/CDN headers instead.
- **AMS / on-prem:** Dispatcher and publish hosts commonly sit behind firewalls that drop Echo. Test with TCP on the real port (`nc -vz host 4503`, `curl -sv --connect-timeout 5 https://host/`), never with ping alone.
- **Hanging deployments, asset uploads or replication between network zones (author → publish over a VPN or interconnect, or Dispatcher → publish through a tunnel) that connect fine but stall on large payloads** are PMTUD black holes until the ICMP path is checked. Replication queues that block with socket timeouts on big packages while small pages replicate are the classic shape.
- **Health checks:** Adobe's load balancers and uptime monitors test HTTP (`/system/probes/health` on AEMaaCS, see [[aemaacs-licensing-content-requests]]), not ICMP. An ICMP-based external monitor on an AEM host measures the firewall's politeness, not AEM.
- **Content-request billing:** ICMP traffic is not HTTP and never counts as a content request. Only `text/html` and `application/json` responses do.

## Cheat sheet

- ICMP = messages *about* packets, carried on IP, no ports, no connections, OS and routers generate it.
- `ping` = Echo Request/Reply. `traceroute` = deliberate Time Exceeded. PMTUD = Fragmentation Needed.
- Header = Type, Code, Checksum (no pseudo-header in v4, pseudo-header in v6), 4 type-specific bytes (Echo id/seq, PMTU next-hop MTU, Redirect gateway), then data.
- An ICMP error quotes the offending packet's IP header + 8 bytes so the sender can match it to a socket; minimal error = 36 ICMP bytes, default echo = 64.
- No error in reply to an error, none for broadcast/multicast, and rate-limited: silence is ambiguous.
- Never block all ICMP. Allow type 3 (especially code 4), type 11, and all ICMPv6 ND + Packet Too Big.
- `sendto: Message too long` is your own kernel, not the network.
- UDP "port open" from `nc`/scanners means only "no Port Unreachable arrived".
- For AEM, ICMP answers "can I reach the edge/host at IP level" and nothing about HTTP, TLS, ACLs, or origin health.
