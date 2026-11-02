# QUIC: the transport under HTTP/3, how a connection is built, what moved out of TCP, and where it stops at the CDN

QUIC (RFC 9000, with RFC 9001 for the TLS binding and RFC 9002 for loss recovery) is a complete transport protocol carried inside UDP datagrams. It is what HTTP/3 (RFC 9114) runs on, the way HTTP/1.1 and HTTP/2 run on TCP + TLS. It is not "UDP with HTTP on top": it re-implements everything TCP did (reliability, ordering, flow and congestion control) plus everything TLS did (handshake, encryption), but per **stream** rather than per connection, in **user space** rather than the kernel, and with every byte after the first packet encrypted. Companion docs: [udp.md](udp.md) (the carrier, and why a QUIC server is one socket demuxed by connection ID), [mtu.md](mtu.md) (the 1200-byte floor), [osi-model.md](osi-model.md) (QUIC is layer 4 inside UDP).

## The many-conversations-on-one-line picture

TCP + TLS is a single phone line: you dial (SYN/SYN-ACK/ACK), then you agree a secret language (TLS handshake), then you talk. Everything said goes through one earpiece in strict order, so if one word is garbled the listener stops and waits for it to be repeated before hearing anything that came after, even words belonging to a different topic. HTTP/2 put many topics on that one line (multiplexed streams) and inherited the stall: one lost packet freezes all streams. That is **head-of-line blocking**, and it was the main motivation for QUIC.

QUIC is a conference line where every topic is its own channel: a lost packet delays only the stream whose bytes were in it, the other streams keep being delivered. The dialling and the secret language happen in the same breath (one round trip, zero on a repeat call), the line follows you when you change phones (connection migration), and the whole conversation, including most of the envelope, is encrypted so the phone company cannot tamper with it (anti-ossification).

## Where it sits

| | TCP + TLS 1.3 + HTTP/2 | QUIC + HTTP/3 |
|---|---|---|
| IP protocol | 6 (TCP) | 17 (UDP) |
| Port | TCP 443 | **UDP 443** (same number, different namespace: a firewall rule for TCP 443 does not open it) |
| Implemented in | kernel (TCP) + library (TLS) | **user space library** (quiche, ngtcp2, msquic, s2n-quic, Java `jdk.internal.net.http.quic` from JDK 26) |
| Handshake | 1 RTT TCP + 1 RTT TLS = **2 RTT** before the first request (1 RTT with TCP Fast Open, rarely deployed) | **1 RTT** combined; **0 RTT** on resumption |
| Streams | HTTP/2 multiplexes inside one byte stream | native to the transport, independently delivered |
| Loss | blocks the whole connection | blocks one stream |
| Encryption | payload only; TCP header in the clear | payload **and** most of the header; only the first byte, the version (long headers) and the destination connection ID are visible |
| Identity of a connection | 4-tuple (src IP, src port, dst IP, dst port) | **connection ID** chosen by each endpoint; survives IP and port changes |
| Discovered by | ALPN `h2` inside the TLS handshake | `Alt-Svc: h3=":443"` header on a prior TCP response, or an `HTTPS` DNS record (RFC 9460) with `alpn=h3`, or a cached hint |

QUIC v1 is RFC 9000; v2 (RFC 9369) is identical in function and exists to exercise version negotiation so middleboxes cannot hard-code v1. Wire versions `h3-29`, `h3-27` seen in `Alt-Svc` headers are IETF draft numbers still advertised for old clients; Google's original gQUIC is gone.

## The handshake

A client that has never talked to the server sends an **Initial** packet: a long-header UDP datagram that carries the TLS 1.3 ClientHello (ALPN `h3`, SNI, key share) as a QUIC CRYPTO frame. Two rules apply to it:

- **It is padded to at least 1200 bytes.** This proves the path carries 1200-byte datagrams (the QUIC minimum, below which the connection must not be used) and limits amplification: a 1200-byte question cannot produce a 60 kB answer.
- **The server may answer at most 3× the bytes it received** until the client's address is validated (the "anti-amplification limit"). If the server's certificate chain is big, it either waits for the client's ACK or sends a **Retry** packet with a token, which costs one extra round trip and is the QUIC equivalent of a SYN cookie. Large certificate chains therefore cost more on QUIC than on TCP; keep them short.

The server replies with its own Initial (ServerHello) plus **Handshake** packets (certificate, finished) and, because the client's key share is already in hand, it can start sending application data right after. The client sends its Finished, and both sides switch to **1-RTT** keys and the **short header** format (one flags byte + destination connection ID + packet number, all but the first byte and the CID encrypted). Total: one round trip to the first HTTP request, where TCP + TLS 1.3 needed two.

**0-RTT** on a repeat visit: the client remembers a session ticket and sends the HTTP request in the very first flight, encrypted with keys derived from the ticket. The server can answer before any handshake completes. The catch is **replay**: an attacker who recorded the 0-RTT datagram can resend it and the server will process it again. RFC 9001 and RFC 8470 therefore say 0-RTT must carry only idempotent requests (GET, HEAD), servers may reject it with `425 Too Early`, and anything with side effects (a POST to a form, a cart update) must wait for 1-RTT. CDNs enforce this; application servers behind a CDN never see 0-RTT because the CDN terminates it.

**Connection IDs** are negotiated during this handshake. Each side gives the other a list of IDs to use; the server demultiplexes incoming datagrams on the destination CID, not the 4-tuple. That is why a load balancer in front of QUIC servers must route on CID (Fastly, Cloudflare and Google encode a server identifier inside the CID for exactly this) and why a client can **migrate**: when its address changes (Wi-Fi to cellular, NAT rebinding), it keeps sending with a new, unused CID, the server does a **path validation** (PATH_CHALLENGE / PATH_RESPONSE) to confirm the new address is not spoofed, and the connection carries on with no re-handshake. TCP would have reset.

## Streams, frames and packets

Three layers inside the UDP payload:

- **Packets** are the unit of encryption and acknowledgement. Each has a **packet number** that is never reused, even for a retransmission, so an ACK is never ambiguous (TCP's retransmission ambiguity, and the Karn's-algorithm workaround for it, do not exist). Packet numbers are protected too, so an observer cannot count packets.
- **Frames** are the content of a packet: STREAM (data for one stream, with offset), ACK (ranges, with the delay the receiver held it), CRYPTO (TLS messages), MAX_DATA / MAX_STREAM_DATA (flow-control credit), NEW_CONNECTION_ID, PATH_CHALLENGE, PING (keepalive), CONNECTION_CLOSE, PADDING. One packet usually carries several frames, and a retransmission re-sends the *frames*, repacked into a new packet, not the old packet.
- **Streams** are ordered byte streams, each with its own flow-control window and its own head-of-line. Stream IDs encode who opened them and whether they are one-way: the low two bits give **client-initiated bidirectional = 0, 4, 8, …; server-initiated bidirectional = 1, 5, 9, …; client unidirectional = 2, 6, …; server unidirectional = 3, 7, …**. HTTP/3 uses a client bidirectional stream per request and unidirectional streams for its control channel and the two QPACK channels.

**Flow control** exists twice: per stream (MAX_STREAM_DATA) and per connection (MAX_DATA), both as absolute byte offsets the sender may not exceed. A receiver that stops reading one stream does not stall the others, which is the mechanism behind "no head-of-line blocking". Receive buffers in user space mean the limits are a library setting, not `net.ipv4.tcp_rmem`.

**Loss recovery and congestion control** (RFC 9002) are TCP's ideas with cleaner inputs: packets are declared lost by packet-number gap (3 packets, like TCP's three duplicate ACKs) or by time (a probe timeout, PTO, roughly a smoothed RTT plus variance); the default congestion controller is NewReno-like with CUBIC and BBR common in deployed stacks. Because ACKs carry the receiver's holding delay, RTT samples are accurate; because packet numbers are unique, a spurious retransmission is detected trivially. ECN is supported from the start.

## What HTTP/3 changes on top

HTTP/3 (RFC 9114) is HTTP/2 semantics moved to QUIC streams: same methods, status codes and header fields, same server push (now dead in practice), same binary framing idea. Two differences matter in practice:

- **QPACK instead of HPACK** (RFC 9204). HPACK's header compression required strict ordering across streams, which QUIC does not give, so QPACK uses two unidirectional streams (encoder, decoder) to synchronise the dynamic table and lets each request choose between "reference the table and risk waiting" or "literal and never block". Default table sizes are small or zero on many CDNs.
- **Discovery is out of band.** A browser cannot start with HTTP/3 unless it knows the server speaks it: it learns from an `Alt-Svc: h3=":443"; ma=86400` header on a previous TCP response (cached per origin for `ma` seconds), from an `HTTPS` DNS record, or from its own cache. Consequence: **the first page load of a session is usually HTTP/2 over TCP, and HTTP/3 kicks in for subsequent requests**; Chrome races a TCP and a QUIC connection and keeps whichever wins. Measurements that only look at the first request miss h3 entirely.

## Firewalls, proxies, NATs

- **Blocking UDP 443 outbound is deliberate and common** in corporate networks because TLS-inspecting proxies cannot see inside QUIC. Browsers notice the silence (no answer in a few hundred ms, or an ICMP Port Unreachable) and fall back to TCP; Chrome then marks QUIC as broken for that network for a while. The user-visible effect is a slower first load and nothing else, which is why "the site is slower from the office" rarely gets attributed to it.
- **An explicit HTTP proxy disables QUIC.** `CONNECT` tunnels carry TCP byte streams; there is no standard way to tunnel UDP through a classic proxy (MASQUE, RFC 9298, is the emerging answer and Chrome supports it for its own proxies). Any client configured with a proxy talks HTTP/1.1 or HTTP/2 through it.
- **NAT mappings for UDP expire in seconds to minutes** ([udp.md](udp.md)). QUIC survives the rebinding through connection migration and path validation, and keeps the mapping alive with PING frames before the idle timeout (negotiated, typically 30 s on CDNs) expires. A client that goes quiet longer than the NAT's timer simply migrates on the next packet.
- **IP fragmentation is forbidden.** QUIC sets DF and uses its own PMTU probing (DPLPMTUD, RFC 8899) upward from 1200; a datagram larger than the path allows is dropped and the stack falls back to 1200. A path that cannot pass 1200-byte UDP datagrams cannot carry QUIC at all, and the client falls back to TCP.
- **Stateless reset.** A server that lost all state (restart, failover to a backend without the CID) answers an unknown CID with a stateless reset token that the client recognises, so the client fails fast instead of waiting for the idle timeout. This is the QUIC equivalent of a TCP RST and the reason a CDN failover is near-invisible.

## Seeing it

```bash
curl -sI https://www.example.net/ | grep -i alt-svc      # "h3=":443"" = server advertises HTTP/3 (over TCP; curl itself used h2 here)
curl --version | grep -i -o 'HTTP3'                      # empty = this curl cannot speak HTTP/3 (most OS-bundled builds)
curl --http3 -sI https://www.example.net/ | head -1      # "HTTP/3 200" on a curl built with ngtcp2 or quiche (Homebrew: brew install curl)
curl --http3-only ...                                    # fail instead of falling back
dig +short HTTPS www.example.net                         # RFC 9460 record; alpn="h3,h2" lets a browser skip the TCP-first visit
sudo tcpdump -i en0 -nn 'udp port 443'                   # long-header Initials ~1200-1250 bytes, then short-header packets; payload opaque
```

In the browser: Chrome DevTools, Network tab, right-click the column header and enable **Protocol**; `h3` means HTTP/3, `h2` means HTTP/2 over TCP. `chrome://net-internals/#quic` (or a `chrome://net-export` log) lists live QUIC sessions and why a QUIC attempt was abandoned. Firefox: `about:networking#http` shows the version per connection.

Wireshark decodes QUIC and HTTP/3 fully if given the TLS keys: set `SSLKEYLOGFILE` for Chrome/Firefox/curl before starting them and point *Preferences → Protocols → TLS → (Pre)-Master-Secret log filename* at it. Without keys you see version, connection IDs, packet type and sizes, nothing else. That is the point of QUIC and the reason network teams dislike it.

## Gotchas

- **"Port 443 is open" means TCP 443.** QUIC needs UDP 443 as a separate rule in every firewall, security group and NACL on the client's egress path. Servers and CDNs also need it inbound.
- **First visit is TCP.** Without an `HTTPS` DNS record the client cannot know about h3 until it has seen an `Alt-Svc` header; single-shot tests (curl, synthetic monitors, load generators) measure HTTP/2 or 1.1, not what repeat visitors get.
- **0-RTT is replayable.** Never let a 0-RTT request have side effects; a CDN that accepts 0-RTT must reject non-idempotent methods or the origin must honour `Early-Data: 1` with `425 Too Early` (RFC 8470).
- **CPU, not bandwidth, is the server-side cost.** User-space packetisation, per-packet encryption, and the absence of mature NIC offloads (UDP GSO/GRO help, but TSO-grade offload for QUIC is new) make QUIC several times more CPU-expensive per byte than TCP + TLS with kernel offload. On a clean low-latency path QUIC can be *slower* than HTTP/2; its wins are on lossy, high-latency and mobile paths. This is why it is a CDN feature, not an origin feature.
- **Load balancers must hash on connection ID**, not on 4-tuple, or connection migration lands on the wrong backend and the session resets. Layer-4 balancers that do not understand QUIC break it quietly under NAT rebinding.
- **Big certificate chains add a round trip** (anti-amplification limit forces a Retry or a wait). Prefer ECDSA leaf + short chain.
- **Large header dynamic tables can reintroduce blocking** through QPACK references; most CDNs keep the table small or zero, which costs compression on header-heavy APIs.
- **tcpdump and most IDS see nothing inside.** Flow-level visibility (bytes, endpoints, timing, the spin bit if enabled) is all that is left; application-level inspection has to happen at a TLS-terminating endpoint, which for QUIC is the CDN or the browser.
- **Idle timeout is not a keepalive.** Both sides advertise `max_idle_timeout`; the connection dies at the lower one unless something (PING) is sent. Long-polling style clients that assume TCP-like hours of idle will be reconnecting.

## What this means for AEM

QUIC and HTTP/3 exist only on the leg **between the visitor's browser and the CDN**. Nothing behind the CDN speaks it, and nothing in AEM needs to change for it.

- **AEM as a Cloud Service.** The Adobe-managed CDN (Fastly) advertises `Alt-Svc: h3=":443"` on its responses, so browsers that have loaded one page switch to HTTP/3 for the rest of the session. The CDN terminates QUIC and opens ordinary TCP + TLS to the Dispatcher; Apache, the Dispatcher module, `access.log`, `X-Forwarded-Proto`, Sling and every `SlingHttpServletRequest` see HTTP/1.1 (or HTTP/2 where the CDN uses it), never HTTP/3. There is no CDN rule, Cloud Manager setting or Dispatcher directive that turns HTTP/3 on or off; it is a platform property. A customer-managed CDN in front of Adobe's offers whatever that CDN supports to the browser and talks TCP to Adobe's edge.
- **Edge Delivery Services.** The same picture: an EDS site served through the aem.live CDN answers with `alt-svc: h3=":443";ma=86400,h3-29=":443";ma=86400` (confirmed from response headers), so the Lighthouse and CrUX numbers that EDS is sold on include HTTP/3 for repeat visitors. Bring-your-own-CDN setups (Cloudflare, Akamai, Fastly in push mode) inherit that CDN's HTTP/3 behaviour.
- **AEM 6.5 on AMS or on-prem.** Apache httpd has no HTTP/3 module (`mod_h2` is HTTP/2 only), and the Dispatcher is an Apache module, so **a Dispatcher can never be the HTTP/3 endpoint**. Getting HTTP/3 to visitors means a CDN or an HTTP/3-capable reverse proxy (nginx 1.25+, HAProxy 2.6+, Caddy) in front of the Dispatcher, with UDP 443 opened to it, and the proxy talking TCP to Apache. AEM's own embedded Jetty is HTTP/1.1 behind the Dispatcher and should never be exposed.
- **AEM calling out.** Java 11 and 17 and 21 `java.net.http.HttpClient` speak HTTP/1.1 and HTTP/2 only; HTTP/3 arrives as an opt-in `HttpClient.Version.HTTP_3` in **JDK 26** (JEP 517), and Apache HttpClient 4.x and 5.x have no HTTP/3 at all. AEMaaCS runs JDK 21 and AEM 6.5 LTS runs on 21 or earlier, so every outbound integration call from AEM (commerce, translation, IMS, webhooks, `HttpClientBuilderFactory`) is TCP. An external API that is HTTP/3-only (none are in practice, since `Alt-Svc` discovery assumes a TCP fallback) would be unreachable; an API fronted by a CDN that *also* offers h3 is simply reached over h2 or 1.1.
- **Performance testing.** JMeter, Gatling, k6 and `ab` speak HTTP/1.1 (k6 and Gatling can do HTTP/2); none speaks HTTP/3. Their numbers describe the CDN's TCP path and are a fair proxy for first-visit latency and origin load, but not for the repeat-visit experience Chrome's field data reports. Use Lighthouse, WebPageTest or CrUX for that, and compare `Protocol` in DevTools when a "fast in the lab, slow in RUM" gap appears.
- **"Slow from one office only."** If a customer reports that an AEMaaCS or EDS site is slow from a corporate network and fine from home, the UDP 443 fallback is a candidate: browsers wait for the QUIC attempt to fail before using TCP, and some proxies make that wait long. `chrome://net-internals/#quic` on an affected machine shows the abandoned sessions. The fix is on their firewall (allow UDP 443, or block it with an ICMP reject rather than a silent drop so the fallback is immediate), not in AEM.
- **Security reviews and WAF.** Rules that inspect request content run at the CDN after QUIC termination, so the AEMaaCS WAF, traffic filter rules and Dispatcher filters apply identically to HTTP/3 and HTTP/2 requests. Content-request billing counts HTTP responses regardless of transport. There is no QUIC-specific attack surface at the origin because the origin has no UDP 443 listener.
- **Advanced networking egress** on AEMaaCS is documented for TCP (HTTP proxy, TCP port forwards); UDP 443 egress is not a thing you can request, which is consistent with the JDK having no HTTP/3 client anyway.

## Cheat sheet

- QUIC = RFC 9000/9001/9002, a full transport (streams, reliability, flow + congestion control, TLS 1.3 built in) inside UDP datagrams on **UDP 443**; HTTP/3 = RFC 9114 on top, QPACK (RFC 9204) for headers.
- Handshake: 1 RTT to first request (TCP + TLS 1.3 = 2), 0 RTT on resumption but replayable, so idempotent requests only; Initial padded to 1200 bytes, server limited to 3× until the address is validated, Retry = SYN cookie.
- Connection identified by **connection ID**, not 4-tuple: survives NAT rebinding and Wi-Fi-to-cellular moves; load balancers must route on CID.
- Streams are independent: a lost packet stalls one stream, not the connection (no head-of-line blocking); stream ID low bits encode initiator and direction (client bidi 0,4,8; server bidi 1,5,9; client uni 2,6; server uni 3,7).
- Packet numbers never repeat, retransmissions re-send frames in new packets; ACKs carry receiver delay; NewReno/CUBIC/BBR in user space.
- Everything after the first byte, version and destination CID is encrypted, including packet numbers: middleboxes and IDS see flows, not requests.
- Discovery: `Alt-Svc: h3=":443"` on a TCP response (so first visit = TCP) or an `HTTPS` DNS record; Chrome races TCP and QUIC.
- Blocked UDP 443 or a configured HTTP proxy = silent fallback to HTTP/2, slower first load; a path that cannot carry 1200-byte UDP datagrams cannot carry QUIC.
- AEM: QUIC ends at the CDN (Adobe-managed CDN and aem.live both advertise h3). Dispatcher/Apache cannot terminate HTTP/3; AEM's JDK (21) and HttpClient have no HTTP/3 client (JDK 26 adds it, opt-in). Load tools measure TCP; RUM measures h3. "Slow from one office" = check UDP 443 on their egress.
