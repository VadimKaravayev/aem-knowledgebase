# MTU: the largest packet a link carries, why 1500, and how to find the path MTU

MTU (Maximum Transmission Unit) is the largest IP packet, in bytes, that a network link will carry in one piece. It is a property of each link and interface, not of a connection. The **path MTU** is the smallest MTU on the whole route. Examples below were run on macOS on 2026-10-25 against `cdn.adobeaemcloud.com` (Fastly). Companion docs: [icmp-explained.md](../network/icmp-explained.md) (Path MTU Discovery and the black hole), [internet-checksum.md](internet-checksum.md), [ip-addresses-explained.md](ip-addresses-explained.md).

## The bridge picture

Every road on a delivery route has a bridge with a height limit. A truck that fits under every bridge gets through. One truck too tall for a single bridge does not, no matter how generous the other roads are. MTU is the bridge height; path MTU is the lowest bridge on the route; and the driver only finds out about a low bridge if someone at that bridge sends word back (ICMP). If nobody does, the truck just never arrives.

## Typical values

| Link | MTU (bytes) | Why |
|---|---|---|
| Ethernet, Wi-Fi | **1500** | The 1980s Ethernet standard, still the default for almost everything |
| Jumbo frames (data-centre LANs, cloud VPC internal) | 9000 (AWS 9001) | Fewer headers per byte; **every device on the segment must agree** |
| PPPoE (many DSL/fibre lines) | 1492 | 8 bytes PPPoE header inside a 1500 frame |
| GRE tunnel | 1476 | 24 bytes GRE + outer IP |
| IPsec / WireGuard / OpenVPN | ~1400 to 1440 | Tunnel headers + encryption overhead; WireGuard default 1420 |
| IPv6 minimum every link must carry | **1280** | Guaranteed floor; IPv4's 576 minimum is only a reassembly guarantee, not a link floor |
| Loopback | 16384 (macOS, verified) or 65536 (Linux) | No wire, no reason for a limit |

## What counts toward it

MTU is the **IP packet**: IP header plus everything inside. The Ethernet frame header and trailer are **not** counted, so a 1500 MTU carries a 1518-byte frame on the wire (1522 with a VLAN tag).

Inside 1500 bytes:

```
  20  IPv4 header (40 for IPv6)
  20  TCP header (more with options; timestamps add 12)
1460  TCP payload  ← MSS (Maximum Segment Size) for IPv4; 1440 for IPv6
```

**MTU is a link property; MSS is what TCP actually negotiates** in the SYN, each side announcing its own interface MTU minus headers. Neither side knows the path. That gap is what Path MTU Discovery exists to close.

For ICMP echo the arithmetic is 20 IP + 8 ICMP + data, so **1472 data bytes is the largest ping that fits in 1500**. Verified: `ping -D -s 1472` to Fastly gets replies; `-s 1473` fails locally with `sendto: Message too long` (the kernel's own interface-MTU check, not the network).

## Why 1500, and why it hasn't changed

A frame must be buffered whole before it can be checksummed and forwarded, so hardware needs an upper bound. Bigger packets mean fewer headers and fewer per-packet CPU interrupts per byte; smaller packets mean a corrupted bit wastes less and one packet hogs a slow link for less time. 1500 was the compromise for 10 Mbit Ethernet in 1980. It has outlived that by forty years because **raising it requires every device on the path to agree**, and the public internet never will. Jumbo frames therefore live only inside domains one organisation controls.

## What happens when a packet is too big for a link

| Case | Router behaviour | Consequence |
|---|---|---|
| IPv4, DF (Don't Fragment) bit clear | **Fragments** the packet; destination host reassembles | Works, but slow path on routers, whole packet lost if one fragment is, many firewalls drop fragments. Verified: `ping -s 2000` without `-D` still gets replies (fragmented), RTT jitter rises |
| IPv4, DF set (**all modern TCP**) | Drops it, sends ICMP **Fragmentation Needed** (type 3 code 4) with the next-hop MTU | Sender shrinks segments and retries: **Path MTU Discovery**. Invisible when it works |
| IPv6 | **Never fragments in transit.** Drops and sends ICMPv6 **Packet Too Big** (type 2) | Only the original sender may fragment, and only if it chooses to |

If the ICMP message is filtered anywhere, the sender never learns the limit, retransmits full-size packets forever, and the connection stalls after a clean handshake. That is the **PMTUD / TCP / MTU black hole**, covered with symptoms and fixes in [icmp-explained.md](../network/icmp-explained.md). macOS has `net.inet.tcp.path_mtu_discovery: 1` on by default; Linux `net.ipv4.ip_no_pmtu_disc=0` likewise.

## Seeing and testing it

```bash
ifconfig en0 | grep mtu                 # macOS interface MTU (verified 1500); ip link show on Linux
ping -D -s 1472 host                    # largest payload for a 1500 path; reply = path MTU ≥ 1500
ping -D -s 1473 host                    # one byte more: "sendto: Message too long" = your own interface MTU
ping -D -s 1400 host                    # step down through a tunnel; silence = too big for some link
sudo tracepath host                     # Linux: hop-by-hop path MTU discovery, no root needed for UDP mode
sudo ifconfig en0 mtu 1400              # macOS: lower the interface MTU (last resort, see below)
```

Rule: a `-D` probe that gets a reply proves path MTU ≥ data + 28. One that times out proves nothing by itself (silence is ambiguous: the host may just ignore ping); step down until replies appear and compare with a known-good host.

## Where it bites

- **VPN client that browses fine but cannot load large pages or finish uploads.** Tunnel MTU lower than the host believes, and ICMP blocked. The most common MTU ticket by far.
- **Cloud VPC on jumbo frames talking to on-prem over a 1500 interconnect.** Everything works inside the VPC, large transfers stall across the link.
- **Container and overlay networks** (VXLAN adds 50 bytes, Docker bridges default 1500): pods reach the internet but big responses hang. Set the overlay MTU to underlay minus overhead.
- **PPPoE home lines** at 1492: an office that works from the data centre but not from staff homes.

Fixes, in order of preference: let ICMP Fragmentation Needed / Packet Too Big through; **MSS clamping** on the tunnel device (rewrites the SYN's MSS so both ends agree up front, TCP only); **TCP MTU probing** on the hosts (`net.ipv4.tcp_mtu_probing=1`, Windows does it by default); lower the interface MTU only as a last resort, since it taxes every connection to fix one path.

## What this means for AEM

- **Replication author → publish, Dispatcher → publish, or asset upload between zones joined by a VPN/tunnel/interconnect that works for small payloads and stalls above a threshold** is an MTU problem until the ICMP path and tunnel MSS are checked. Replication queues blocked with socket timeouts on large packages while small pages flow is the signature.
- **AEMaaCS** customers never touch MTU: the CDN, Adobe's network and advanced-networking egress are Adobe's. The exception is the customer side of a **dedicated egress / VPN** (advanced networking): that tunnel's MTU and MSS clamping are the customer's to get right, and symptoms land on the AEM side as "integration calls time out only for large responses".
- **AMS / on-prem:** jumbo frames between author, publish and the data store are fine inside one network; the first 1500 hop outward (CDN origin pull, SMTP, external APIs) must still pass ICMP or be MSS-clamped.
- Verified: the public path from here to Adobe's Fastly edge carries full 1500-byte packets with DF set, so a stall on an AEMaaCS site from a client is almost always the **client's** VPN or tunnel, not Adobe's edge.

## Cheat sheet

- MTU = max IP packet per link, in bytes; path MTU = smallest on the route. 1500 Ethernet, 1492 PPPoE, ~1400 to 1440 VPNs, 1280 IPv6 floor, 9000 jumbo (LAN only).
- Excludes the Ethernet frame header; includes IP + TCP headers. MSS = MTU − 40 (IPv4) / − 60 (IPv6). Largest 1500-fitting ping = 1472 data bytes.
- Too big: IPv4 fragments unless DF is set; all modern TCP sets DF; IPv6 never fragments in transit. Drop + ICMP (type 3 code 4 / ICMPv6 type 2) = PMTUD.
- Filtered ICMP → black hole: handshake works, large transfers hang, no error anywhere.
- `sendto: Message too long` is your own interface MTU, not the path.
- Fix order: pass the ICMP, clamp MSS on the tunnel, TCP MTU probing on hosts, lower interface MTU last.
