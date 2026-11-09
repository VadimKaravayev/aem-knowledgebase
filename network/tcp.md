# TCP: how a connection works and how to read one in Wireshark

TCP (Transmission Control Protocol, RFC 9293, which consolidates RFC 793 and its fixes) is the transport under HTTP, TLS, SSH, SMTP and most of what a server talks. It turns IP's unreliable packets into a reliable, ordered, flow-controlled **byte stream** between two sockets. This doc explains the mechanics by what you see on the wire, then how Wireshark presents them. Companion docs: [udp.md](udp.md) (the contrast), [quic.md](quic.md) (TCP rebuilt on UDP), [tcpdump.md](tcpdump.md) (capturing), [mtu.md](mtu.md) (where MSS comes from), [internet-checksum.md](internet-checksum.md), [netcat.md](netcat.md).

## The phone-call picture

UDP is a postcard; TCP is a phone call. You dial (SYN), the other side picks up and says hello (SYN-ACK), you say hello back (ACK), and only then does anyone speak. Each side numbers every byte it says so the other can tell what went missing and ask for it again, and each side tells the other how much it can take in before it needs a pause (the window). At the end both say goodbye (FIN) or one slams the phone down (RST).

Three consequences that drive everything below:

- **A connection is a shared state machine**, identified by the 4-tuple (source IP, source port, destination IP, destination port). Each end keeps sequence numbers, windows and timers for it; nothing in the middle has to (which is why NATs and firewalls that *do* keep state can break it by forgetting).
- **Bytes, not messages.** `send()` boundaries are not preserved. Two writes may arrive as one read or one write as five. Any protocol on top (HTTP) must frame its own messages (`Content-Length`, chunking).
- **Every guarantee costs round trips.** Setup is 1 RTT before the first byte of data, 2 RTT with TLS 1.2, 1 RTT with TLS 1.3 on top (QUIC merges them). Throughput is bounded by window ÷ RTT regardless of link speed.

## Where it sits and what the header holds

IP protocol number 6. Header is 20 bytes minimum, up to 60 with options; the **data offset** field says how long it is, in 32-bit words (5 to 15).

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-------------------------------+-------------------------------+
|          Source port          |       Destination port        |
+---------------------------------------------------------------+
|                        Sequence number                        |
+---------------------------------------------------------------+
|                    Acknowledgement number                     |
+-------+-----+-+-+-+-+-+-+-+-+-+-------------------------------+
| Data  |Rsvd |C|E|U|A|P|R|S|F|          Window               |
|offset |     |W|C|R|C|S|S|Y|I|                                 |
|       |     |R|E|G|K|H|T|N|N|                                 |
+-------+-----+-+-+-+-+-+-+-+-+-+-------------------------------+
|           Checksum            |        Urgent pointer         |
+-------------------------------+-------------------------------+
|                    Options (0 to 40 bytes)                    |
+---------------------------------------------------------------+
```

- **Source port.** Chosen by the connecting side's kernel from the ephemeral range (macOS 49152–65535, Linux 32768–60999). It is the client's half of the connection identity and the only thing that distinguishes two connections from the same client to the same server. A busy client (a reverse proxy, a load generator) can exhaust the range when old connections linger in `TIME_WAIT` (below). **Server sockets do not get a new port per connection**: the server answers from port 443 for every client, and the kernel demultiplexes on the full 4-tuple (see the table in [udp.md](udp.md)).
- **Destination port.** The service. Well-known ports are a convention, not a protocol feature; Wireshark guesses the upper protocol from the port and is wrong for anything on a non-standard port (use *Decode As*).
- **Sequence number (32-bit).** The byte offset, within this sender's stream, of the first payload byte in this segment. **It counts bytes, not segments.** It starts at a random **Initial Sequence Number (ISN)** per connection, per direction. `SYN` and `FIN` each "consume" one sequence number even though they carry no data, which is why the first data byte after a SYN with ISN *x* is numbered *x* + 1. The field wraps at 2³²; at 10 Gbit/s that happens every ~3.4 s, which is what the timestamp option's PAWS check (RFC 7323) protects against.
- **Acknowledgement number (32-bit, valid when ACK is set).** The next byte the sender of this segment *expects* to receive: "I have everything up to here minus one". It is **cumulative**: an ACK for 5000 covers all earlier bytes, and a lost ACK is harmless if a later one arrives. ACK is set on every segment after the first SYN; a "pure ACK" is a segment with ACK set and no payload.
- **Flags.** `SYN` synchronise sequence numbers (handshake only), `ACK` acknowledgement number is valid, `FIN` sender has no more data, `RST` abort / "no such connection", `PSH` deliver to the application now (set on the last segment of a write; modern stacks deliver immediately anyway, so it is mostly a hint about application write boundaries), `URG` urgent pointer valid (effectively unused), `ECE`/`CWR` explicit congestion notification (RFC 3168). Wireshark renders them as a string such as `·······S·` and `·A··S··` as well as flag names in brackets: `[SYN]`, `[SYN, ACK]`, `[PSH, ACK]`, `[FIN, ACK]`, `[RST]`.
- **Window (16-bit).** How many more bytes the sender of this segment is willing to receive beyond the acknowledgement number. Flow control, from the receiver. 16 bits caps it at 65535, which is why window scaling exists (below).
- **Checksum.** Mandatory, over a pseudo-header (IPs, protocol, length), the header and the data; recipe in [internet-checksum.md](internet-checksum.md). Captures taken *on the sending host* show it wrong because the NIC computes it after the capture point (offload), see Gotchas.
- **Options.** Each is kind, length, value; kind 1 (`NOP`) pads to 4-byte alignment, kind 0 ends the list. The ones that matter:

| Option | Kind | Where | What |
|---|---|---|---|
| MSS | 2 | SYN only | Largest payload *this sender will accept* per segment. Default 536 if absent. 1460 on a 1500-byte IPv4 path, 1440 on IPv6, lower on VPN/PPPoE/tunnelled paths. Not negotiated: each side uses the smaller of its own and the peer's. The classic MTU black hole ([mtu.md](mtu.md)) is "handshake fine, first full-size segment never arrives". |
| Window scale | 3 | SYN only | Shift count 0–14 applied to this sender's own window field in all later segments. Both SYNs must carry it or neither side scales. |
| SACK permitted | 4 | SYN only | "I can receive SACK blocks." |
| SACK | 5 | Any ACK | Up to 4 (3 with timestamps) ranges of bytes received beyond the cumulative ACK. |
| Timestamps | 8 | SYN then every segment | `TSval` (my clock) and `TSecr` (echo of yours). Gives per-segment RTT measurement, PAWS wrap protection, and lets Wireshark compute RTT on any ACK. |

MSS, window scale and SACK-permitted **can only be set during the handshake**. If the SYN pair is not in the capture, Wireshark cannot know the scale factor and prints `Window size scaling factor: -1 (unknown)`; `-2` means the handshake *was* seen and scaling was not negotiated.

## The three-way handshake

```
client 192.0.2.10:51234                       server 203.0.113.5:443
   |-- SYN  seq=ISNc  win=65535 [MSS=1460 WS=6 SACK_PERM TS] -->|   client: SYN_SENT
   |<- SYN,ACK seq=ISNs ack=ISNc+1 win=65160 [MSS=1460 WS=7 SACK_PERM TS] --|  server: SYN_RECEIVED
   |-- ACK  seq=ISNc+1 ack=ISNs+1 win=2058 (×64 = 131712) --------------->|  both: ESTABLISHED
   |-- PSH,ACK seq=ISNc+1 ack=ISNs+1 len=517  (TLS ClientHello) --------->|
```

1. **SYN.** The client picks an ISN, states its MSS, its window scale shift, whether it accepts SACK, and its timestamp. The window in this segment is *unscaled* (the scale applies only after the handshake), so a 65535 here does not mean a 64 KB buffer. No payload (TCP Fast Open, RFC 7413, is the exception and rare outside Google-ish stacks).
2. **SYN-ACK.** The server picks *its own* ISN, acknowledges ISNc + 1 (the SYN consumed one number), and lists the options **it** supports. Options are not echoed: a server that omits `WS` or `SACK_PERM` turns the feature off for the connection. The server is now in `SYN_RECEIVED`, holding an entry in its **SYN queue** (not yet a socket the application can `accept()`).
3. **ACK.** The client acknowledges ISNs + 1. The connection is `ESTABLISHED` on both ends; the server moves it from the SYN queue to the **accept queue** (its size is the `listen()` backlog, capped by `net.core.somaxconn`, 4096 on current Linux). Data can ride in this third segment; in practice the TLS ClientHello or HTTP request follows in the next one a few microseconds later.

**Why random ISNs.** Sequence numbers are the only thing that makes a segment "belong" to a connection, so a predictable ISN lets an off-path attacker inject RSTs or data. RFC 6528 specifies a per-4-tuple keyed hash plus a clock, which is what Linux and macOS do. The other reason is to keep stale segments from a previous incarnation of the same 4-tuple from being accepted (also what `TIME_WAIT` is for).

**What a failed handshake looks like**, and this is the first thing to read in any "cannot connect" capture:

| You see | Means |
|---|---|
| SYN, then SYN again at +1 s, +2 s, +4 s… nothing back | Silently dropped: firewall, security group, wrong route, host down, or the server's SYN queue is full and SYN cookies are off. Linux retries 6 times (`tcp_syn_retries`), ~127 s to `ETIMEDOUT`; the application's connect timeout usually fires first. |
| SYN → RST, ACK immediately | Host reachable, **nothing listening on that port** (or a firewall that rejects rather than drops). `Connection refused`. |
| SYN → ICMP unreachable | Router or host firewall rejected it; see [icmp.md](icmp.md). |
| SYN → SYN-ACK, SYN-ACK repeated, client never ACKs | Asymmetric path: the SYN-ACK is being dropped on the way back, or the client capture point is not where the client is (NAT, proxy). |
| SYN → SYN-ACK → ACK → data → RST | Connected at layer 4, then the *application* or a proxy killed it: TLS alert, idle timeout, backend down behind a load balancer that still accepts. |

**SYN cookies.** When the SYN queue overflows (a SYN flood, or an application that stopped calling `accept()`), Linux with `tcp_syncookies=1` (the default) stops storing state and instead encodes the MSS and a hash into the ISN of the SYN-ACK; the third ACK proves the client is real. Side effect visible in Wireshark: window scale and SACK are lost for those connections unless timestamps are on, and `netstat -s | grep -i cookie` counts them. A server "sending SYN cookies" is the kernel telling you the application is not keeping up with new connections.

## Data transfer: sequence arithmetic, ACKs, retransmission

Every segment covers the byte range `[seq, seq + len)`. The next sequence number from that sender is `seq + len` (+1 for SYN or FIN). Wireshark exposes this as `tcp.nxtseq`, and the receiver's ACK should eventually equal it. Reading a stream is just checking that each side's `Ack=` keeps catching up with the other side's `Seq+Len`.

- **Delayed ACK.** Receivers acknowledge every second full segment or after a timer (Linux 40 ms minimum, macOS and Windows up to 200 ms) rather than per segment, so "one ACK per two data segments" is normal, not loss.
- **Nagle and delayed ACK.** A sender with Nagle on (the default) holds a small write until the previous one is acknowledged; a receiver with delayed ACK waits up to 200 ms to acknowledge. Two small writes in a row (HTTP header then body as separate `write()` calls) can therefore stall 40–200 ms per request. The cure is `TCP_NODELAY` or writing header and body together; the symptom in a capture is a data segment, then a gap of exactly the delayed-ACK timer, then the next.
- **Retransmission.** The sender keeps unacknowledged data and retransmits it when the **RTO** (retransmission timeout, initial 1 s per RFC 6298, then smoothed from measured RTT, Linux floor 200 ms) expires, backing off exponentially. Faster: **fast retransmit** after 3 duplicate ACKs (the receiver re-ACKs the same number for every out-of-order segment it gets), or RACK-TLP (RFC 8985, Linux default) which uses timestamps instead of counting dup ACKs. A retransmission in Wireshark is a segment whose `seq` is below the highest `nxtseq` already seen from that sender.
- **Congestion control** (slow start, CUBIC, BBR) decides how much *unacknowledged* data the sender keeps in flight (`cwnd`). **It is not in any header**; you infer it from `tcp.analysis.bytes_in_flight` growing and collapsing. Initial window is 10 segments (RFC 6928), so a 14 KB response fits in the first flight and a 15 KB one costs another RTT.

## Window and window scaling: flow control

The window field announces how much buffer the *receiver* has free. The sender may have at most `window` bytes unacknowledged; when the receiver's application is slow to read, the advertised window shrinks toward **zero window**, the sender stops, and the receiver sends a **window update** when space frees up (the sender also probes with 1-byte **zero-window probes** so a lost update does not deadlock).

Throughput ceiling: **bytes per second ≤ window ÷ RTT**. A 65535-byte window on a 100 ms path is 5.2 Mbit/s no matter how fast the links are. That is why RFC 7323 adds the **window scale** option: a shift count *s* (0–14) sent in each side's SYN; afterwards every window value from that side means `value × 2^s`. Maximum window becomes 65535 × 2¹⁴ ≈ 1 GiB. Each direction has its own factor, chosen by the kernel from its maximum receive buffer (Linux derives it from `net.ipv4.tcp_rmem[2]`, 6 MB by default → shift 7 = ×128; macOS typically ×64; Windows ×256). Wireshark shows three fields: `tcp.window_size_value` (the 16-bit number on the wire), `tcp.window_size_scalefactor` (the multiplier), and `tcp.window_size` (the product, labelled "Calculated window size"). The handshake above shows `win=2058` on the wire meaning 131712 bytes.

What breaks it:

- **Middleboxes that strip or zero the WS option** (old firewalls, some NAT appliances and SD-WAN boxes). One side scales and the other does not, so the receiver believes the sender's windows are 128 times smaller than they are, or vice versa: transfers crawl or stall after the first ~64 KB. The tell in Wireshark is a client-side capture showing `WS=128` in its own SYN but the server-side capture showing no WS in the arriving SYN.
- **Missing handshake in the capture.** Wireshark then reports windows unscaled and may flag false `[TCP Window Full]`. Preferences → Protocols → TCP → "Scaling factor to use when not available from capture" lets you set the factor by hand once you know it.
- **Zero window from the server** means the *server application* is not reading: a thread pool exhausted, a GC pause, a blocked request. From the client it means the browser or client is the bottleneck. Either way the network is innocent.
- **`[TCP Window Full]`** means the *sender* has filled the receiver's window and must wait: receive-buffer limited. More buffer (`tcp_rmem`, `SO_RCVBUF`) or a faster reader, not more bandwidth.

## SACK: selective acknowledgement

Cumulative ACK has a weakness: if bytes 1000–1999 are lost but 2000–9999 arrive, the receiver can only keep saying "ack 1000". The sender does not know whether just one segment is missing or everything after it, and without SACK it either retransmits the whole window or one segment per RTT. **SACK (RFC 2018)** adds an option on the ACK listing ranges already received beyond the cumulative point: `Ack=1000, SACK: 2000–9999`. The sender retransmits exactly 1000–1999 and nothing else.

- Enabled only if **both** SYNs carried `SACK_PERM`. All current stacks do; a connection without it today means a middlebox stripped the option or a SYN cookie rebuilt the connection without it.
- Up to 4 blocks per ACK (3 when timestamps are on, because the option space is 40 bytes). Blocks are advisory: a receiver may still discard SACKed data under memory pressure (reneging), so the sender keeps it until cumulatively acknowledged.
- **D-SACK (RFC 2883):** a first block *below* the ACK number reports "you sent me this twice". It tells the sender its retransmission was spurious (the original was merely delayed), so it can undo its congestion-window cut. Wireshark labels these `[TCP D-SACK]` and the retransmission `[TCP Spurious Retransmission]`.
- A burst of ACKs with growing SACK ranges and the same `Ack=` is the fingerprint of **a single lost segment on a fat pipe**; the gap closes with one fast retransmission. Many small SACK blocks in a row point to reordering or a lossy link (Wi-Fi, a saturating queue), not a single drop.

## Head-of-line blocking

TCP promises an *ordered* byte stream, so the receiving kernel may hand bytes to the application only up to the first gap. If segment 1000–1999 is lost and 2000–9999 arrive, those 8 KB sit in the receive buffer (SACKed, acknowledged in an option) while `recv()` returns nothing. The application sees a stall of at least one RTT (fast retransmit or RACK), or a full RTO (initial 1 s, Linux floor 200 ms) when the loss is at the tail of a burst and nothing follows to trigger duplicate ACKs. Once the retransmission fills the gap, everything buffered is released at once. SACK makes the *sender* resend only the gap, but it does nothing for delivery order: the data is already there and the application still cannot have it.

This is harmless when the stream carries one thing. It hurts when one connection carries several independent things:

- **HTTP/1.1 pipelining** has head-of-line blocking at the application layer: responses must come back in request order, so one slow response holds back the fast ones. Browsers never enabled pipelining and open about 6 parallel connections per origin instead.
- **HTTP/2** fixes the application-layer problem by interleaving many streams on one connection, then runs into the transport-layer problem: TCP sees one byte stream, so a single lost segment carrying part of one image stalls *every* stream until it is retransmitted. On a clean path HTTP/2's single connection wins. On a lossy one (mobile, congested Wi-Fi, a few percent loss) it can do worse than HTTP/1.1's six connections, where one loss stalls only a sixth of the traffic.
- **TLS records** add a second layer. A record (up to 16 KB) can be decrypted only once all its segments have arrived, so one lost segment holds back the whole record even after TCP has delivered the bytes around it. Servers that start with small records (about one MSS) and grow them after the first flight ("dynamic record sizing") limit this for time-to-first-byte.

How it shows up: in Wireshark, a `[TCP Retransmission]` or `[TCP Fast Retransmission]` followed immediately by the application consuming a large block (the receive window jumps back up). In browser DevTools on HTTP/2, several unrelated requests on the same connection stall and then complete together.

What helps, in order of effect: **QUIC/HTTP/3**, which orders per stream so a loss delays only the stream whose bytes were in the lost packet; **several TCP connections** instead of one multiplexed one; **less loss** on the path; and tail-loss probing (TLP in RACK-TLP, the Linux default), which turns a tail loss's full RTO into roughly two RTTs. Bigger buffers and more bandwidth do not help, because the blocked data has already arrived.

The term comes from switching, where the packet at the front of an input queue blocks the ones behind it even though their output port is free. In TCP the "line" is the receive buffer in front of the application.

## Teardown: FIN, RST and TIME_WAIT

```
active closer                                        passive closer
   |-- FIN,ACK seq=x  ------------------------------>|  FIN_WAIT_1 → CLOSE_WAIT
   |<- ACK     ack=x+1 ------------------------------|  FIN_WAIT_2
   |<- FIN,ACK seq=y  -------------------------------|  (passive side calls close())  LAST_ACK
   |-- ACK     ack=y+1 ----------------------------->|  TIME_WAIT  → CLOSED after 2×MSL
```

- **FIN means "I have no more to send"**, not "the connection is closed". The other direction stays open (half-close), which is how `nc` can send a file and still receive the reply. Each FIN consumes a sequence number and must be acknowledged. The passive side's ACK and FIN are often one segment, giving a 3-segment close.
- **TIME_WAIT** is held by the side that sent the first FIN, for 2 × MSL (Linux 60 s fixed; macOS 2 × 15 s = 30 s, `net.inet.tcp.msl`). It exists so that a delayed segment from this connection cannot be mistaken for the next connection on the same 4-tuple, and so the final ACK can be re-sent if the peer retransmits its FIN. Thousands of `TIME_WAIT` sockets are normal on a busy client or proxy; they only hurt when they exhaust the ephemeral port range toward one destination (about 28 000 ports ÷ 60 s ≈ 470 new connections/s to one server on Linux without keep-alive). `tcp_tw_reuse` (client side, timestamps required) is the knob; `tcp_tw_recycle` was removed from Linux as broken behind NAT.
- **CLOSE_WAIT** piling up means **the local application received a FIN and never called `close()`**: a leaked `HttpClient` response, a servlet that did not consume the body. It is a bug in the process that owns the socket, not in the network, and `lsof -nP -iTCP -sTCP:CLOSE_WAIT` names the process.
- **RST** is the abort path: sent for a segment that matches no connection (the "connection refused" to a SYN; a data segment arriving after the peer rebooted or a stateful firewall forgot the flow), by `close()` on a socket with unread received data, or by `SO_LINGER` with zero timeout. No ACK, no retransmission, no `TIME_WAIT`. **An RST in the middle of a working stream is almost always a middlebox idle timeout**: cloud load balancers (AWS NLB 350 s, Azure LB 4 min default, GCP 10 min), corporate firewalls (often 300–3600 s), NAT tables. The connection sat idle in a client pool, the box forgot it, the next request got an RST, and the client logged `Connection reset by peer` or `NoHttpResponseException`.
- **Keep-alive** (`SO_KEEPALIVE`) sends a 1-byte-behind probe after idle (Linux 7200 s default, far too long for middleboxes; set `tcp_keepalive_time` or the client library's idle validation lower than the middlebox timeout). Wireshark marks probes `[TCP Keep-Alive]` and the response `[TCP Keep-Alive ACK]`. This is TCP keep-alive, unrelated to HTTP `Connection: keep-alive`.

## Seeing it in Wireshark

Capture with `tcpdump -w` ([tcpdump.md](tcpdump.md)) or Wireshark itself; prefer a *capture filter* (`tcp port 8080`) to keep the file small, and *display filters* to read it. Field names below are the display-filter names (Wireshark 4.x).

**Stream index (`tcp.stream`).** Wireshark's own 0-based counter for each TCP conversation (4-tuple) in order of first appearance **in this file**. It is not on the wire, it restarts from 0 for every capture, and it changes if you save a filtered subset and reopen it. A new SYN on a 4-tuple whose previous connection Wireshark saw close gets a **new** stream index, so one `(client:port → server:443)` pair can appear as several streams when the client recycles ports. Use it everywhere: right-click a packet → *Follow → TCP Stream* sets the filter `tcp.stream eq N` and shows the two directions' payload in red/blue; *Statistics → Conversations → TCP* lists every stream with byte counts, duration and which side started it; `tshark -z conv,tcp -q -r f.pcap` is the same table on the CLI.

**Relative vs raw sequence numbers.** By default Wireshark subtracts each direction's ISN, so the SYN shows `Seq=0`, the first data byte `Seq=1`, and `Ack=` values are byte counts you can read at a glance. The real ISNs are still there as `tcp.seq_raw` and `tcp.ack_raw`, and the preference *Protocols → TCP → Relative sequence numbers* turns the subtraction off. You need raw numbers when **correlating two captures of the same connection** (client side and server side, both sides of a NAT, a firewall log that quotes sequence numbers): relative numbers differ if one capture missed the handshake, raw numbers are identical on both sides of anything that is not a proxy. A proxy or TLS-terminating load balancer produces *two* connections with unrelated ISNs, which is how you prove one exists.

**Reading the handshake in the Info column:**

```
51234 → 443 [SYN]      Seq=0 Win=65535 Len=0 MSS=1460 WS=64 TSval=... TSecr=0 SACK_PERM
443 → 51234 [SYN, ACK] Seq=0 Ack=1 Win=65160 Len=0 MSS=1460 SACK_PERM TSval=... TSecr=... WS=128
51234 → 443 [ACK]      Seq=1 Ack=1 Win=131712 Len=0 TSval=... TSecr=...
```

`Win=` on the two SYNs is raw; from the third segment Wireshark prints the scaled value. `WS=64` is the multiplier (the option carries shift 6). `tcp.analysis.initial_rtt` is computed from SYN → ACK and attached to every packet of the stream, so `tcp.analysis.initial_rtt > 0.1` finds all connections with a slow handshake. `tcp.completeness` (3.6+) is a bitmask of what the file holds for the stream: 1 SYN, 2 SYN-ACK, 4 ACK, 8 data, 16 FIN, 32 RST; `tcp.completeness == 31` is a textbook complete connection, `tcp.completeness & 1 == 0` finds streams whose start you missed.

**Expert analysis (`tcp.analysis.*`).** With *Analyze TCP sequence numbers* on (default) Wireshark annotates segments in square brackets and colours them black. `tcp.analysis.flags` shows every annotation; `tcp.analysis.flags && !tcp.analysis.window_update` is the everyday "show me trouble" filter.

| Annotation / filter | Meaning | Usually caused by |
|---|---|---|
| `tcp.analysis.retransmission` | Same data sent again after RTO | Loss, or ACKs lost on the return path |
| `tcp.analysis.fast_retransmission` | Resent after 3 dup ACKs | A single lost segment |
| `tcp.analysis.spurious_retransmission` | Resent data the receiver had already ACKed | Delayed ACK or reordering, sender RTO too tight |
| `tcp.analysis.duplicate_ack` | Receiver re-ACKing the same number | It got something out of order; look for the gap |
| `tcp.analysis.lost_segment` ("Previous segment not captured") | Gap in seq from that sender | Real loss **or a capture drop**: if the receiver never dup-ACKs, the capture missed it, not the network |
| `tcp.analysis.out_of_order` | Lower seq than expected, not a retransmission by timing | Multipath, LAG, reordering middlebox |
| `tcp.analysis.zero_window` / `window_full` / `window_update` | Flow control (see above) | Receiver application slow / receive buffer small |
| `tcp.analysis.keep_alive` | 1-byte probe | Idle pool connection being validated |
| `tcp.analysis.ack_rtt` | Time between a segment and the ACK that covers it | Per-segment RTT; graph it |
| `tcp.analysis.bytes_in_flight` | Unacked bytes from this sender | Proxy for cwnd; flat ceiling = window limited |
| `tcp.time_delta` | Gap since previous packet **in this stream** (enable *Calculate conversation timestamps*) | `tcp.time_delta > 1` finds every stall |

**Other filters worth memorising:**

```
tcp.flags.syn==1 && tcp.flags.ack==0          # every connection attempt
tcp.flags.syn==1 && tcp.flags.ack==1          # every accepted one; compare counts
tcp.flags.reset==1                            # aborts; who sent them?
tcp.flags.fin==1                              # who closed first
tcp.port==8080 && tcp.len>0                   # payload only, no pure ACKs
tcp.options.wscale.shift                      # SYNs carrying window scale
!tcp.options.sack_perm && tcp.flags.syn==1    # SYNs without SACK: stripped?
tcp.options.sack_le                           # ACKs carrying SACK blocks
tcp.window_size_scalefactor == -1             # streams whose handshake is missing
tcp.seq_raw == 0x1a2b3c4d                     # correlate with a firewall log
http.request || tls.handshake.type == 1       # first application bytes per stream
```

**Graphs.** *Statistics → TCP Stream Graphs → Time/Sequence (tcptrace)* for one direction: the I-bars are segments, the lower step line is the ACKs, the upper green line is the receiver's advertised window edge. Segments hugging the green line = receive-window limited; a flat stretch = stall (check what the ACK side did); vertical red marks = SACK/retransmission. *Round Trip Time* and *Window Scaling* graphs are the other two tabs. *Statistics → I/O Graphs* with `tcp.analysis.retransmission` as a second series shows whether loss tracks load.

**Decoding what rides on top.** Non-standard ports: right-click → *Decode As* → TCP port → HTTP. TLS: Wireshark shows only `Client Hello`/`Server Hello`/`Application Data` unless the browser or JVM wrote pre-master secrets to a file (`SSLKEYLOGFILE` in Chrome/Firefox/curl; `-Djavax.net.debug` does not produce it, a JVM needs a key-logging Java agent) and *Protocols → TLS → (Pre)-Master-Secret log filename* points at it. HTTP/1.1 is reassembled across segments when *Allow subdissector to reassemble TCP streams* is on (default), so one HTTP response may be shown on the frame that carries its last byte.

**tshark equivalents** for scripting:

```bash
tshark -r f.pcap -q -z conv,tcp                                   # conversation table
tshark -r f.pcap -Y 'tcp.analysis.retransmission' | wc -l         # count retransmissions
tshark -r f.pcap -Y 'tcp.flags.syn==1 && tcp.flags.ack==0' -T fields -e tcp.stream -e ip.dst -e tcp.dstport
tshark -r f.pcap -Y 'tcp.stream==7' -T fields -e frame.time_relative -e tcp.seq -e tcp.ack -e tcp.len -e tcp.flags.str
```

## Gotchas

- **Bad checksums on everything you sent.** Capturing on the sending host records the segment before the NIC fills in the checksum (offload). Wireshark's checksum validation is off by default for this reason; if someone turned it on, "Checksum: incorrect" on outbound-only packets is the offload, not corruption. Incoming bad checksums are real.
- **Segments bigger than the MSS** (8 KB, 64 KB "packets") in a host capture are TSO/GSO: the kernel handed one giant buffer to the NIC which cut it into MSS-sized segments on the wire. Inbound, LRO/GRO glues them back together. Counts of segments, ACK-per-segment ratios and "Len=" values from a host capture are therefore not what the wire carried; `ethtool -K eth0 tso off gso off gro off` on Linux, or capture on a mirror port or the other end, when exact segment behaviour matters.
- **"Previous segment not captured" is not proof of network loss.** If the receiver's next ACK covers the gap, the bytes arrived and the *capture* dropped the packet (check `tcpdump`'s "dropped by kernel" line). Only dup ACKs or a retransmission confirm the network lost it.
- **Window scaling cannot be inferred mid-stream.** No handshake in the file means every window value is unscaled and `[Window Full]` may be a false alarm. Start captures *before* opening the connection, or set the factor manually.
- **The same 4-tuple can be several streams, and the same stream index means nothing across files.** Filter by `tcp.stream`, but quote the 4-tuple and raw ISN when handing a finding to someone else.
- **A SYN retransmitted at +1 s, +3 s, +7 s, +15 s** (Linux doubling) with nothing in between is a drop, not slowness. Nothing you do on the client changes that; the answer is on the firewall path.
- **RST from the server with `Seq=1 Ack=…` right after your request** is the application or a proxy, not the kernel: a TLS alert follow-up, a load balancer with no healthy backend, a proxy enforcing a protocol check. An application-level rejection is an HTTP 4xx over a healthy connection, so an RST means something in front of the web server.
- **Delayed-ACK + Nagle stalls** look like a 40 ms or 200 ms gap after every small write. If the gap equals the receiver OS's delayed-ACK timer, it is this, not the network.
- **Wireshark's port-based protocol guess** leaves traffic on non-standard ports undecoded or mislabelled; use *Decode As* before concluding "no HTTP in this capture".

## What this means for AEM

- **AEM as a Cloud Service gives you no capture point.** There is no shell or NIC access on Dispatcher or AEM containers, and the CDN terminates the visitor's TCP. What you can capture is the browser side (DevTools *Timing*: "Initial connection" is the handshake, "Waiting" is the server) and systems that AEM calls or that call AEM. On AMS or on-prem 6.5 you can run `tcpdump` on the Dispatcher and publish hosts, and that is where the rest of this doc applies.
- **Dispatcher `connectTimeout` vs `renderTimeout`** are two different wire events: `connectTimeout` is waiting for the publish's SYN-ACK (repeated SYNs in a capture mean network or a dead render; an immediate RST means nothing listening on that port), `renderTimeout` is waiting for the first byte after the request was sent (handshake complete, so the render is up but slow).
- **Pooled HTTP connections from AEM** (`HttpClientBuilderFactory`, replication agents) are exposed to the mid-stream idle-timeout RST described under Teardown: intermittent `Connection reset` or `NoHttpResponseException` on the first call after a quiet period, fixed by validating or evicting idle connections below the middlebox's limit.

## Cheat sheet

- TCP = IP protocol 6, 20–60-byte header, reliable ordered byte stream per **4-tuple**; server reuses its port, client's ephemeral source port is the per-connection half.
- **Seq** = byte offset of first payload byte (starts at a random **ISN** per direction; SYN and FIN each consume one); **Ack** = next byte expected, cumulative. Next seq = seq + len (+1 for SYN/FIN).
- **Handshake**: SYN (ISNc, MSS, WS, SACK_PERM, TS) → SYN-ACK (ISNs, ack=ISNc+1, server's own options) → ACK (ack=ISNs+1). Options are set **only** here. SYN with no reply = dropped (retries at 1, 2, 4, 8… s); SYN → RST = nothing listening.
- **Window** = receiver's free buffer in bytes, 16-bit; **window scale** shift 0–14 from each SYN multiplies that side's later windows (×128 Linux default). Throughput ≤ window ÷ RTT. Zero window = slow reader; Window Full = sender limited by receiver.
- **SACK** (needs `SACK_PERM` in both SYNs) names received ranges beyond the ACK so only the gap is resent; D-SACK reports a duplicate. Absence today = middlebox or SYN cookie.
- **Head-of-line blocking**: a lost segment holds back all later bytes in the receive buffer until it is retransmitted (at least 1 RTT, a full RTO on tail loss). SACK doesn't help with this. It stalls every HTTP/2 stream on the connection and a whole TLS record. Fixes: QUIC (per-stream ordering), several connections, less loss. Not bandwidth or buffers.
- **Teardown**: FIN each way, active closer holds **TIME_WAIT** 60 s (Linux) / 30 s (macOS); CLOSE_WAIT pile = local app never closed; mid-stream **RST = idle-timeout middlebox** nine times in ten.
- **Wireshark**: `tcp.stream` is Wireshark's per-file conversation counter (Follow TCP Stream, Conversations); Seq/Ack are **relative** by default, raw in `tcp.seq_raw`/`tcp.ack_raw` for cross-capture correlation; `tcp.window_size_value` (wire) × `tcp.window_size_scalefactor` = `tcp.window_size`, factor `-1` = handshake not captured; `tcp.analysis.flags && !tcp.analysis.window_update` for trouble, `tcp.time_delta > 1` for stalls, `tcp.analysis.initial_rtt` per stream, `tcp.completeness` for what the file holds; tcptrace graph for window-limited vs loss.
- **Host-capture artefacts**: bad outbound checksums (offload), >MSS segments (TSO/GRO), "previous segment not captured" with no dup ACK (capture drop).
- **AEM**: no capture point inside AEMaaCS; Dispatcher `connectTimeout` = waiting for SYN-ACK, `renderTimeout` = waiting for first byte; pooled `HttpClient` connections hit idle-timeout RSTs.
