# The OSI model: seven layers, the four that actually exist on the wire, and how to use it for troubleshooting

The OSI model (ISO 7498, 1984) is a reference vocabulary: seven numbered layers, each providing a service to the one above and using the one below. Nobody runs OSI protocols; everybody runs TCP/IP, which was designed before the model with four layers. The value of OSI today is the *numbers*: "L2 switch", "L3 firewall", "L4 load balancer", "L7 WAF" are shorthand everyone understands. Companion docs in this folder each live at one layer: [mac-addresses.md](mac-addresses.md) and [arp.md](arp.md) at 2, [ip-addresses.md](ip-addresses.md), [icmp.md](icmp.md) and [mtu.md](mtu.md) at 3, [udp.md](udp.md) at 4, [dhcp.md](dhcp.md) at 7.

## The envelope-in-an-envelope picture

A letter (your data) goes into an envelope with a name on it (the application protocol), which goes into a courier pouch labelled with a door number (the port), which goes into a mail sack labelled with a street address (the IP), which is loaded onto a truck with a licence plate (the MAC) on a road with a speed limit and a maximum load (the physical link). Each worker along the way reads only their own label and never opens the inner envelope. The truck is swapped at every depot; the sack travels end to end; the courier pouch is only opened at the destination; the letter is read by the recipient.

That is encapsulation, and the one picture answers most layer questions: *which label does this device read, and which labels does it rewrite?*

## The seven layers

| # | Layer | Unit | Identifier | What it does | Examples | Device that works here |
|---|---|---|---|---|---|---|
| 7 | Application | message | URL, hostname, method | The protocol the program speaks | HTTP, DNS, SMTP, TLS handshake content, SSH, DHCP | Web server, proxy, WAF, L7 load balancer, CDN |
| 6 | Presentation | | | Encoding and representation: character sets, compression, serialisation | TLS *is usually filed here*, gzip, JSON/ASN.1 | (none; it lives in libraries) |
| 5 | Session | | | Dialogue control: open/close/resume a conversation | TLS session resumption, RPC, NetBIOS | (none) |
| 4 | Transport | segment (TCP) / datagram (UDP) | **port** | Process-to-process delivery; TCP adds reliability, ordering, flow and congestion control; UDP adds only ports | TCP, UDP, QUIC, SCTP | L4 load balancer, stateful firewall, NAT |
| 3 | Network | packet | **IP address** | Host-to-host delivery across networks: addressing, routing, fragmentation, TTL | IPv4, IPv6, ICMP, IPsec, routing protocols' payloads | Router, L3 switch, packet-filter firewall |
| 2 | Data link | frame | **MAC address** | Delivery on one link: framing, local addressing, error detection (FCS), access control | Ethernet, Wi-Fi (802.11), PPP, VLAN tags (802.1Q), ARP sits between 2 and 3 | Switch, bridge, access point, NIC MAC block |
| 1 | Physical | bit / symbol | | Signalling on the medium: voltages, light, radio, connectors, line coding | 1000BASE-T, 802.11ax radio, fibre, DOCSIS | Cable, transceiver, hub, repeater, NIC PHY |

Three things the table hides:

- **Layers 5 and 6 are empty in practice.** TCP/IP folds them into the application. TLS is the usual argument: it encrypts (6), negotiates sessions (5), and is carried as TCP payload like any application (7). The honest answer is "between 4 and 7"; many people say "L6" or just "L7". No device is "an L5 device". If an exam or an RFP asks, put TLS at 6 and session resumption at 5 and move on.
- **The network stack ends at 4.** The kernel implements 1 through 4 (with the NIC doing 1 and most of 2). Everything from 5 up is a library linked into the process, which is why a browser, `curl` and a JVM can each handle TLS differently on the same host while all of them share one IP stack.
- **ARP, ICMP and routing protocols don't fit.** ARP carries MAC and IP addresses but is not inside IP, so it is "2.5". ICMP is carried *by* IP but is part of the network layer. BGP runs over TCP (so layer 7 by carriage) yet its job is to program layer 3. The model describes services, not packaging; when the two disagree, say which you mean.

## The TCP/IP model, which is what you are actually running

| TCP/IP layer | OSI layers | What people call it |
|---|---|---|
| Application | 5, 6, 7 | "L7" |
| Transport | 4 | "L4" |
| Internet | 3 | "L3" |
| Link | 1, 2 | "L2" (and "L1" for the cable) |

Use OSI numbers for talking and the TCP/IP layering for thinking. Nothing in a packet capture, a kernel, or an RFC is organised by OSI layers.

## Encapsulation on the wire

A typical HTTPS request on Ethernet, with the byte offsets a capture tool shows:

```
offset  0   Ethernet header  14 bytes   dst MAC, src MAC, EtherType 0x0800 (IPv4)     L2
offset 14   IPv4 header      20 bytes   src IP, dst IP, protocol 6 (TCP), TTL, length L3
offset 34   TCP header       20+ bytes  src port, dst port 443, seq, ack, flags       L4
offset 54   TLS record       5 bytes    content type, version, length                 "L6"
offset 59   application data            HTTP request, encrypted                       L7
            Ethernet FCS      4 bytes   CRC-32, checked and stripped by the NIC        L2
```

The MTU of 1500 counts offsets 14 onward and excludes the Ethernet header and FCS ([mtu.md](mtu.md)): the limit is on the *packet*, not the frame. Each header carries a field that names the next layer: EtherType names the L3 protocol, the IP protocol number names L4 (6 TCP, 17 UDP, 1 ICMP), the port names the L7 service by convention. That chain is how a receiver demultiplexes without any out-of-band agreement, and it is the same chain `tcpdump` filter words follow: `ether`, `ip`, `tcp`, `port` ([tcpdump.md](tcpdump.md)).

**What each device rewrites:**

| Device | Reads | Rewrites | Never touches |
|---|---|---|---|
| Switch | dst MAC | nothing | IP and above |
| Router | dst IP, TTL | **both MACs** (new frame per link), TTL, IP header checksum | ports and payload |
| NAT | IP + port | src or dst IP **and** port, both checksums | payload (except ALGs) |
| L4 load balancer | IP + port, connection state | dst IP/port (and often src) | payload |
| Reverse proxy, CDN, WAF, Dispatcher | the whole request | **everything**: terminates TCP and TLS, opens a *new* connection to the origin | (the original packet does not exist on the other side) |

The last row is the one that matters for web work: beyond an L7 proxy, the origin sees the proxy's IP, the proxy's TCP options, the proxy's TLS version, and only the HTTP headers the proxy chose to forward (`X-Forwarded-For`, `Via`, `Forwarded`). Everything below L7 was reconstructed.

## Using the model to troubleshoot

The model earns its keep as a checklist that stops you from testing layer 7 when layer 1 is broken. Bottom-up, with the test that isolates each layer:

| Layer | Question | Test | Typical failure |
|---|---|---|---|
| 1 | Is there a link? | `ifconfig` shows `RUNNING` / `status: active`; link LED | Unplugged cable, bad SFP, Wi-Fi not associated ([ifconfig.md](ifconfig.md)) |
| 2 | Can I reach the next hop? | `arp -a` has the gateway's MAC; `ping <gateway>` | Wrong VLAN, duplicate MAC, ARP blocked, isolated client mode |
| 3 | Can I reach the destination IP? | `ping <ip>`, `traceroute -I <ip>` | No route, wrong subnet mask, firewall drop, PMTU black hole ([icmp.md](icmp.md), [traceroute.md](traceroute.md)) |
| 3.5 | Does the name resolve? | `dig name`, `dig +tcp name` | DNS blocked (UDP 53 but not TCP), wrong resolver, stale cache ([udp.md](udp.md)) |
| 4 | Is the port open? | `nc -vz host 443`, `curl --connect-timeout 5 -sv` up to "Connected" | Service down, listening on another address, stateful firewall, NAT timeout |
| 5/6 | Does TLS complete? | `openssl s_client -connect host:443 -servername host`, `curl -v` up to "SSL connection using" | Expired cert, SNI mismatch, protocol version, intercepting proxy |
| 7 | Does the application answer correctly? | `curl -sv` full exchange, status code and headers | 403 from ACL, 502 from origin, wrong vhost, cache |

Two readings of "top-down" are also useful. **Divide and conquer** starts at the layer most likely to be broken from the symptom: "connects but hangs on large responses" is L3 (PMTU), not L7; "works from one office, not another" is L3/L4 (allow list, UDP 443 blocked) before L7. **Follow the identifier**: if the problem is tied to a hostname it is L7 or DNS, to an IP it is L3, to a port it is L4, to one machine on a LAN it is L2.

## The things people get wrong

- **"Layer 3 switch" is a router** with a switch's port density; it routes between VLANs in hardware. "Layer 2 switch" is the plain kind.
- **A firewall's layer tells you what it can see.** A packet filter (L3/L4) matches on IPs and ports and cannot read a hostname or URL; blocking a *site* on it means blocking its current IPs. A WAF (L7) reads the request and can block a path or a header, but has to terminate TLS to do so. "The firewall blocks ICMP by port" is a category error: ICMP has no ports.
- **An L4 load balancer preserves the client IP; an L7 one does not** unless it adds `X-Forwarded-For` or uses the PROXY protocol. IP allow lists checked at the origin behind an L7 proxy see the proxy, not the client.
- **TLS is not "layer 7 security"** in the sense of protecting the application: it protects the transport between two hops. An L7 proxy in the middle decrypts, reads and re-encrypts; the origin's TLS is a separate hop.
- **MTU is an L2-defined limit on an L3 object.** The link decides the maximum frame; the limit everyone quotes (1500) is the IP packet that fits inside ([mtu.md](mtu.md)).
- **HTTP/3 breaks the "HTTP is on TCP" assumption.** QUIC is a transport (L4) carried in UDP; HTTP/3 rides on it. An L4 rule that allows "web = TCP 80/443" blocks it; browsers fall back to TCP after a timeout.
- **"Layer 8" is the user.** Jargon for a problem in the person or the process, not the network. Harmless, but do not put it in a report.

## What this means for AEM

- **Where each AEM-adjacent component sits.** CDN, WAF, Dispatcher and Apache are L7 proxies: each terminates the connection and opens a new one, so the publish instance sees the Dispatcher's IP and only the headers it forwards. Cloud Manager's IP allow list is an L3 identifier (client IP) checked at L7 (an HTTP 403 from the CDN), which is why a blocked office still pings and completes TLS. Custom domains and TLS certificates in Cloud Manager configure the CDN's TLS; the certificate never reaches the publish JVM, which is why no AEM-side keystore or Jetty setting is involved.
- **Troubleshooting order for "the site is down".** Resolve the name, reach the CDN edge at L3 (`ping`, `traceroute -I`), connect at L4 (`nc -vz host 443`), complete TLS (`openssl s_client -servername`), then read the HTTP status and the CDN cache headers. A 5xx with a CDN `Server` header and a `Via` line is an L7 answer from the edge about the origin; a connection timeout is below L7 and no AEM log will show it.
- **AMS / on-prem** is where the lower layers are yours: VLANs between Dispatcher and publish, MTU on VPN links between author and publish (replication that stalls on large packages is L3 PMTU, not an AEM bug), a packet filter that opens TCP 4503 but not UDP 53 so replication fails on name resolution. On AEMaaCS layers 1 to 4 belong to Adobe and the only knobs you have are L7 (Dispatcher config, CDN rules) plus the IP allow list and advanced-networking egress at L3/L4.
- **Load balancers in front of publish** (AMS and self-hosted): L4 keeps client IPs for AEM but cannot do sticky sessions by cookie; L7 can, and must forward `X-Forwarded-For` if anything in AEM (geo, rate limiting, custom logging) is to see the real client; `request.log` records the TCP peer, so behind an L7 balancer that is the balancer.

## Cheat sheet

- Seven layers, numbered from the cable up: Physical, Data link, Network, Transport, Session, Presentation, Application. Real stacks have four: Link (1+2), Internet (3), Transport (4), Application (5+6+7).
- Identifiers: MAC at 2, IP at 3, port at 4, hostname/URL at 7. Units: frame, packet, segment/datagram, message.
- Devices: switch 2, router 3, NAT and L4 balancer 4, proxy/CDN/WAF/Dispatcher 7. A router rewrites MACs; an L7 proxy rebuilds everything.
- Each header names the next: EtherType → IP protocol number → port → application.
- Layers 5 and 6 exist only on paper; file TLS at 6 if forced. ARP is 2.5, ICMP is 3, QUIC is 4 inside UDP.
- Troubleshoot bottom-up with one test per layer (link, ARP, ping, dig, nc, s_client, curl), or jump to the layer the symptom names (hostname → 7/DNS, IP → 3, port → 4, one LAN host → 2).
- Beyond an L7 proxy the origin sees the proxy, not the client, unless headers are forwarded; the allow list that returns 403 is L3 data enforced at L7.
