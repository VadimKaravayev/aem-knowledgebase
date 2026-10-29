# UDP: datagrams, the 8-byte header, ports as the demultiplexing key, and what "no reply" means

UDP (User Datagram Protocol, RFC 768) is the thin transport: it adds ports and an optional checksum to IP and nothing else. No handshake, no acknowledgements, no retransmission, no ordering, no congestion control, no stream. Each `send()` becomes exactly one datagram that arrives whole, arrives out of order, or does not arrive at all, and nobody tells you which. Companion docs: [icmp.md](icmp.md) (the only error channel UDP has), [traceroute.md](traceroute.md) and [dhcp.md](dhcp.md) (two UDP users), [internet-checksum.md](internet-checksum.md), [mtu.md](mtu.md).

## The postcard picture

TCP is a phone call: you dial, the other side picks up, you both know the line is open, and if a sentence is garbled you repeat it. UDP is a stack of postcards: each one is addressed and dropped in the box on its own. Postcards can arrive in any order, a few get lost, and the post office never tells you which. If you need to know a card arrived, the *recipient* has to write back, and that reply is just another postcard.

Everything that follows is a consequence:

- **The application owns reliability.** DNS retries after a timeout and falls back to TCP; NTP averages many samples; QUIC and RTP implement their own acknowledgements, ordering and congestion control on top of UDP. "UDP is unreliable" means the *transport* promises nothing, not that the protocols built on it are.
- **Message boundaries are preserved.** One `send()` = one `recv()`. TCP would merge or split them into a byte stream. This is why datagram sizes matter (below) and why UDP is the natural fit for request/response and periodic-sample protocols.
- **No reply has at least three meanings**: closed port, dropped by a firewall, or the server simply does not answer that input. UDP cannot distinguish them; only an ICMP Port Unreachable coming back says "closed" for sure.

## Where it sits and what the header holds

UDP is IP protocol number 17 (TCP is 6, ICMP is 1). The header is 8 bytes, fixed, no options:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-------------------------------+-------------------------------+
|          Source port          |       Destination port        |
+-------------------------------+-------------------------------+
|            Length             |           Checksum            |
+-------------------------------+-------------------------------+
|                        Data (0 or more bytes)                 |
```

- **Source port (2 bytes).** Where the reply should go. May be `0` meaning "no reply expected", which some one-way protocols use. Normally an ephemeral port picked by the kernel (macOS 49152–65535, Linux 32768–60999).
- **Destination port (2 bytes).** The service. Same 16-bit space as TCP, but a separate namespace: TCP 53 and UDP 53 are different sockets, and a firewall rule for one does not open the other.
- **Length (2 bytes).** Header *plus* data, in bytes, so the minimum is 8. It is redundant with the IP length field; IPv6 jumbograms set it to 0 and rely on IP.
- **Checksum (2 bytes).** Ones-complement sum over a pseudo-header (source and destination IP, protocol, UDP length), the UDP header and the data. **Optional on IPv4: a sender may write `0` to mean "not computed"** (a real computed zero is sent as `0xFFFF`). **Mandatory on IPv6.** Covering the IP addresses in the sum is why a NAT must recompute it when it rewrites addresses. Recipe: [internet-checksum.md](internet-checksum.md).

That is the whole protocol. Every other TCP feature (sequence numbers, windows, flags, options) has no counterpart.

## Multiplexing and demultiplexing: what the ports are for

IP delivers a packet to a *host*. Dozens of processes on that host are talking to the network at once, so the transport layer has to do two things: **multiplex** on the way out (merge every process's traffic into one stream of IP packets, stamping each with a source port so replies can be sorted back) and **demultiplex** on the way in (read the destination port and hand the payload to the right socket). For UDP, ports are the *entire* job of the protocol beyond the checksum. In the postcard picture the port is the apartment number on the envelope: the postal service only reads the street address, the building's mailroom reads the apartment.

**Outbound (mux).** When a process sends on an unbound UDP socket, the kernel picks an ephemeral source port, records that the socket owns it, and writes it into the header. Different sockets, different source ports; the same socket keeps its port for its lifetime. That number is the only thing that gets the reply back to the right process, which is why a UDP client that expects an answer must keep the socket open, and why "fire and forget" senders can set source port 0.

**Inbound (demux).** This is where UDP and TCP differ in a way that shapes every server design:

| | Demux key | Sockets on a server | What `recv` gives you |
|---|---|---|---|
| **UDP** | destination IP + destination port (**2-tuple**) | **One socket per port**, shared by every peer | Payload + the peer's address from `recvfrom()`; the application sorts peers itself |
| **TCP** | source IP + source port + destination IP + destination port (**4-tuple**) | One listening socket plus **one socket per connection** | A byte stream from exactly one peer |

Consequences of the 2-tuple rule:

- **A UDP server is one socket and one queue.** A DNS server answers the whole internet from a single socket on port 53; every client's datagram lands in the same receive buffer, and the server reads the source address and port off each one to know where to reply. There is no `accept()`. This is why one slow reader stalls *all* peers (the silent full-buffer drop below), and why high-throughput UDP servers either use `SO_REUSEPORT` to open several sockets on the same port and let the kernel hash incoming 4-tuples across them (how multi-worker DNS and QUIC servers scale), or spread load across ports.
- **Two clients behind the same NAT look like two source ports**, not two addresses. NAPT is itself a multiplexing layer: it rewrites the source port so that many private hosts share one public IP, and demultiplexes replies by that port. The mapping timeout described under "Sockets without connections" is this table expiring.
- **`connect()` narrows the filter to a 4-tuple** without creating a new socket: the kernel then drops datagrams from any other peer and can route ICMP errors to the socket. It is demux by 4-tuple done on request.
- **Specific beats wildcard.** A socket bound to `192.0.2.10:53` receives in preference to one bound to `0.0.0.0:53` on the same host; `lsof -nP -iUDP` shows the difference as `192.0.2.10:53` versus `*:53`. Binding to `*` and then wondering why a daemon on a specific address "steals" the traffic is a classic.
- **Demux failure is the only error UDP produces.** A datagram whose destination port has no socket is counted (`dropped due to no socket`) and, if the host is polite, answered with ICMP Port Unreachable ([icmp.md](icmp.md)). That is the entire basis of UDP traceroute and of UDP port scanning.

**Port demux is not enough, so protocols add their own.** Because one socket serves many exchanges, UDP protocols carry a second-level identifier inside the payload: DNS has a 16-bit transaction ID, NTP matches on the transmit timestamp, SNMP has a request-id, RTP has SSRC, DHCP has `xid`. The 2008 Kaminsky DNS cache-poisoning attack worked because the kernel accepted any datagram to the resolver's port and the 16-bit ID was the only remaining check; the fix (RFC 5452) was to **randomise the source port per query** so an attacker must guess 32 bits, which is a demux decision used as a security control. Check a resolver with `dig +short porttest.dns-oarc.net TXT` style services, or watch `tcpdump` for a resolver that always uses the same source port.

**QUIC drops the 4-tuple altogether.** Because a mobile client's IP and NAT port change mid-session, QUIC packets carry a **connection ID** and the server demultiplexes on that, not on the address. A connection survives the client moving from Wi-Fi to cellular; the same trick over TCP is impossible because the kernel's 4-tuple demux would treat the new address as a different connection. This is also why a QUIC server behind a load balancer needs the balancer to hash on connection ID, not on the 4-tuple, or migrated connections land on the wrong backend.

## Sizes: how big a datagram can be

- **Theoretical maximum:** 65535 bytes of Length, so 65527 bytes of data; with a 20-byte IPv4 header the practical ceiling is **65507** data bytes.
- **What fits without IP fragmentation:** the path MTU minus the IP and UDP headers. On a 1500-byte Ethernet path that is **1472** bytes over IPv4 and **1452** over IPv6 (40-byte header). For the IPv6 guaranteed minimum MTU of 1280 bytes it is 1232, which is exactly the EDNS buffer size the DNS community settled on for "safe over UDP".
- **Bigger datagrams are IP-fragmented**, and UDP has no way to avoid it other than not sending them (there is no MSS negotiation). Lose one fragment and the whole datagram is gone: the receiver cannot reassemble, never delivers it, and the sender gets no error. Many firewalls and some cloud load balancers drop non-initial fragments outright because they carry no port numbers to filter on. This is the single most common cause of "UDP works for small messages and silently fails for big ones", and the reason DNS over UDP capped answers at 512 bytes for decades, then at an EDNS-advertised size, and sets the **TC (truncated) bit** so the client retries over TCP when an answer still does not fit.

Rule of thumb for anything you build on UDP: keep datagrams **at or under 1200 bytes** unless you control every link on the path. QUIC's spec uses the same number for its initial packets.

## Sockets without connections

A UDP socket is just a local port with a queue. Two details catch people:

- **`connect()` on a UDP socket sends nothing.** It records a default destination, makes the kernel drop incoming datagrams from any other address, and, crucially, **is what makes ICMP errors reach the application**. On Linux an *unconnected* UDP socket never sees `ECONNREFUSED` for a Port Unreachable because the kernel cannot know which `sendto()` it belongs to; a *connected* one does. A UDP client that "hangs instead of failing fast" is often simply unconnected.
- **The receive queue is finite and overflow is silent.** When datagrams arrive faster than the application reads them, the kernel discards the excess and increments a counter; nothing is sent back and the sender keeps going. The counter is the only evidence:

```bash
netstat -s -p udp            # macOS: "dropped due to full socket buffers", "dropped due to no socket"
ss -s; nstat -az UdpRcvbufErrors UdpInErrors     # Linux
```

"Dropped due to no socket" counts datagrams for ports nobody is listening on (these are what trigger ICMP Port Unreachable, when the host sends it at all). "Dropped due to full socket buffers" is a slow consumer: raise `net.core.rmem_max` / `SO_RCVBUF` or make the reader faster.

## Who uses it

| Port | Protocol | Why UDP |
|---|---|---|
| 53 | DNS | One small question, one small answer; a handshake would triple the cost. Falls back to TCP on truncation or for zone transfers. |
| 67/68 | DHCP | Client has no IP yet, so broadcast datagrams are the only option. [dhcp.md](dhcp.md). |
| 123 | NTP | Timing needs the lowest, most predictable latency; retransmission would corrupt the measurement. |
| 161/162 | SNMP | Polling and traps; lost samples are tolerable. |
| 514 | syslog (classic) | Fire-and-forget log shipping; lines are lost silently when the collector is down. TLS syslog uses TCP 6514. |
| 5353 / 1900 | mDNS, SSDP | Multicast discovery on the LAN; TCP cannot multicast. |
| 33434+ | traceroute probes | Deliberately sent to closed ports so the destination answers with Port Unreachable. [traceroute.md](traceroute.md). |
| 443 | **QUIC / HTTP/3** | A whole transport (reliability, streams, congestion control, TLS 1.3) rebuilt on UDP to avoid TCP head-of-line blocking and middlebox ossification. |
| 51820, 500/4500 | WireGuard, IPsec IKE/NAT-T | VPN tunnels: the inner traffic has its own TCP, so an outer TCP would only add retransmission on retransmission. |
| 8125 | StatsD | Metrics from app to local agent; losing a counter beats blocking the app. |
| RTP/RTCP (dynamic) | Voice, video, WebRTC | A late packet is worthless; retransmission is harmful. |

Two properties follow from "no handshake" and explain a lot of firewall policy:

- **Spoofable.** Nothing proves the source address, so an attacker can send a small request with a victim's address and have the server send a large reply to the victim. DNS, NTP (`monlist`), SSDP and memcached have all been **amplification** vectors. This is why cloud providers block inbound UDP by default, why public resolvers rate-limit, and why QUIC servers answer an unknown client with a Retry token before doing real work.
- **Stateless for NAT.** A NAT or stateful firewall has no FIN to watch, so it keeps a UDP mapping alive on a timer, typically **30 seconds to a few minutes**, versus hours for established TCP. Long-lived UDP sessions must send keepalives (WireGuard's `PersistentKeepalive = 25` exists for exactly this), and a reply that arrives after the mapping expired is dropped silently.

## Seeing it

```bash
sudo tcpdump -i en0 -nn 'udp port 53'            # capture; filter grammar in tcpdump.md
sudo tcpdump -i en0 -nn 'udp and (ip[6:2] & 0x1fff != 0 or ip[6] & 0x20 != 0)'   # UDP fragments only
lsof -nP -iUDP                                    # which process owns which UDP port (macOS/Linux)
ss -lun                                           # Linux listeners
netstat -an -p udp                                # macOS listeners (no process names without root + lsof)
dig +notcp +bufsize=1232 example.net A            # force UDP with a 1232-byte EDNS buffer
dig +tcp example.net A                            # force TCP: if this works and UDP does not, UDP 53 is filtered
nc -u host 514                                    # interactive UDP sender (type a line, it goes out as one datagram)
```

`nc -u -z` deserves its own warning, verified on macOS:

```
$ nc -u -z -w 2 127.0.0.1 9
Connection to 127.0.0.1 port 9 [udp/discard] succeeded!
```

Nothing listens on port 9 there. `nc` reports "succeeded" for UDP whenever no Port Unreachable reached it before it gave up, and on macOS it does not wait at all with `-z`, so **the result is meaningless for UDP**. `nmap -sU` is more careful (it reads the ICMP reply and reports `open|filtered` when there is none) but is slow for the same reason: it has to wait out the silence per port. Trust only an application-level check: `dig` for DNS, `sntp`/`ntpdate -q` for NTP, `snmpget` for SNMP.

## Gotchas

- **"UDP port open" from a scanner means "no ICMP came back".** See above. Combine with the [icmp.md](icmp.md) silence table before concluding anything.
- **UDP-mode traceroute ends in stars at CDNs and hardened hosts** that drop the probes without replying. Re-run with `-I` or TCP before calling the path broken. [traceroute.md](traceroute.md).
- **Large datagrams vanish, small ones work.** IP fragments dropped on the path. Check with `dig +bufsize=512` versus `+bufsize=4096`, or `ping -D -s <n>` to find the path MTU ([mtu.md](mtu.md)), and keep payloads near 1200 bytes.
- **No congestion control.** A UDP sender that floods will drown its own TCP traffic on the same link, and the loss shows up elsewhere first (SSH stalls, HTTPS timeouts). Anything that sends sustained UDP must pace itself or use QUIC.
- **Checksum `0` on IPv4 is legal; on IPv6 it is a discard.** A middlebox or tunnel that zeroes UDP checksums breaks IPv6 UDP silently. [internet-checksum.md](internet-checksum.md).
- **Unconnected UDP sockets never get `ECONNREFUSED`.** Clients that should fail fast on a closed port must `connect()` first (Java: `DatagramSocket.connect()` has the same effect).
- **NAT mappings time out in seconds, not hours.** Idle UDP "connections" through a NAT stop working; keepalives are mandatory.
- **A UDP service that stops reading does not refuse, it drops.** Watch the kernel drop counters; the sender sees nothing.

## What this means for AEM

AEM itself speaks only TCP: HTTP on 4502/4503 (and 80/443 at the Dispatcher), replication over HTTP(S), JDWP, JMX/RMI, SMTP, LDAP 389/636, SAML and OAuth over HTTPS, TarMK cold standby on 8023. There is no UDP listener in a stock instance, and a firewall that opens only the TCP ports is correct. UDP appears at the edges:

- **DNS (UDP 53) sits in front of every outbound call AEM makes**: replication agents, HttpClient integrations, SMTP, LDAP, translation and commerce connectors, IMS token exchanges. A firewall that allows TCP 443 egress but not UDP 53 produces `UnknownHostException` in `error.log` for every integration while `curl` from an allowed host works. Test with `dig` on the AEM host, and with `dig +tcp` to see whether only the UDP side is blocked. The JVM additionally caches lookups (`networkaddress.cache.ttl`, `networkaddress.cache.negative.ttl` in `java.security`), so a fixed firewall can still need a restart before AEM sees it.
- **NTP (UDP 123) decides whether SAML, OAuth and IMS work.** SAML assertions carry `NotBefore`/`NotOnOrAfter` windows measured in minutes, IMS service-credential JWTs carry `exp`, and S3/Azure data-store signatures have clock tolerances. An AMS or self-hosted host whose NTP is blocked drifts until logins fail with "assertion expired" or "invalid timestamp" while HTTP is perfectly healthy. Check `ntpq -p` / `chronyc tracking` / `sntp -d <server>` before touching the IdP configuration. On AEMaaCS Adobe owns the clocks.
- **Syslog (UDP 514) and SNMP (UDP 161/162) are the AMS/on-prem log-shipping and monitoring transports.** Lost lines under load are expected with UDP syslog; if completeness matters use TCP or TLS syslog. Neither exists on AEMaaCS, where logs come through Cloud Manager and the Splunk/log-forwarding integration over HTTPS.
- **HTTP/3 over QUIC (UDP 443) is live at the edge.** Adobe's managed CDN for AEMaaCS answers with `alt-svc: h3=":443"`, and so does aem.live for Edge Delivery Services (confirmed from the response headers). Browsers that have talked to the site once switch to UDP 443 for later requests. Consequences: a corporate firewall that blocks outbound UDP 443 makes browsers fall back to HTTP/2 over TCP after a timeout, which looks like a slow first load from one office only; the CDN terminates QUIC and talks TCP to the Dispatcher and origin, so nothing in your Dispatcher config, Apache logs or AEM ever sees HTTP/3; and a load test tool speaking only HTTP/1.1 is not measuring what Chrome measures. For a self-hosted CDN in front of AMS, HTTP/3 is a CDN feature to enable, not something Apache or AEM provide.
- **AEMaaCS advanced networking is TCP.** Flexible port egress and VPN port forwards are documented for HTTP proxying and TCP destination ports (the port-forward example in Adobe's docs declares `"protocol": "TCP"`); UDP egress is not documented. Design integrations that need UDP (custom NTP, syslog to a collector, SNMP traps, StatsD) out of AEMaaCS, or put a TCP/HTTPS collector in between. DNS from AEMaaCS is resolved by Adobe's infrastructure, not through your port forwards.
- **Content-request billing and CDN logs** count HTTP responses only; the protocol underneath (QUIC or TCP) does not change what is counted.

## Cheat sheet

- UDP = IP protocol 17, 8-byte header: source port, destination port, length (header + data, min 8), checksum (pseudo-header included; optional on IPv4, mandatory on IPv6).
- Ports are the whole job: mux outbound by source port, demux inbound by **destination IP + port only (2-tuple)**, so a UDP server is one socket shared by all peers (vs TCP one socket per 4-tuple connection); `connect()` narrows to 4-tuple, `SO_REUSEPORT` spreads one port over workers, NAT muxes hosts onto ports, DNS adds a transaction ID + random source port on top, QUIC demuxes on a connection ID so sessions survive address changes.
- One `send()` = one datagram; arrives whole, out of order, or not at all; nobody is told. Reliability, ordering and congestion control are the application's job (DNS retries, QUIC rebuilds TCP).
- Max data 65507 (IPv4); 1472 fits a 1500 MTU unfragmented; keep it near 1200 to survive fragment-dropping paths. A lost fragment loses the whole datagram silently.
- The only "closed" signal is ICMP type 3 code 3 (Port Unreachable). Silence = closed, filtered, or ignored; `nc -u -z` says "succeeded" regardless (verified on macOS, even for a closed loopback port).
- Receive-queue overflow and no-listener drops are silent; read them from `netstat -s -p udp` / `nstat`.
- Unconnected sockets never see `ECONNREFUSED`; `connect()` a UDP socket to get errors.
- Spoofable (amplification) and stateless (NAT mappings expire in 30 s to minutes, keepalives required).
- AEM is TCP-only; UDP matters for DNS before every outbound call, NTP for SAML/IMS clocks, syslog/SNMP on AMS, and HTTP/3 between browsers and the CDN (which the Dispatcher never sees). AEMaaCS egress is documented for TCP only.
