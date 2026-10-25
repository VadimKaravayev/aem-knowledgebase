# tcpdump: capturing packets, reading the one-line format, and what it can and cannot tell you about AEM

`tcpdump` captures packets from an interface (or reads them from a file) and prints one line per packet, optionally with the bytes. It is the tool for "what is actually on the wire" when logs and curl disagree. Everything below was verified on macOS with tcpdump 4.99 / libpcap 1.10, Oct 2026, using a capture whose addresses are documentation values. Companion docs: [ifconfig.md](ifconfig.md) (which interface), [icmp.md](icmp.md), [arp.md](arp.md), [mtu.md](mtu.md).

## Before anything: root, and the macOS surprise

Capturing needs raw access to the interface, so `sudo` (or membership of the `access_bpf` group that Wireshark's installer creates on macOS). Without it:

```
tcpdump: en0: You don't have permission to capture on that device
((cannot open BPF device) /dev/bpf0: Permission denied)
```

**macOS gotcha (verified):** even `tcpdump -d 'tcp port 4502'`, which only compiles the filter and prints the BPF program without capturing, fails without root with `ioctl(SIOCIFCREATE): Operation not permitted`, because Apple's build creates a `pktap` pseudo-interface first. To test a filter without root, compile it against a file: `tcpdump -d -r any.pcap 'tcp port 4502'`. Reading files never needs root.

Listing interfaces does not need root and tells you which are up and associated:

```
$ tcpdump -D
1.ap1 [Up, Running, Wireless, Not associated]
2.en0 [Up, Running, Wireless, Associated]
3.awdl0 [Up, Running, Wireless, Associated]
5.utun0 [Up, Running]
```

## The commands you actually type

```bash
sudo tcpdump -i en0 -nn                                   # everything on en0, numeric hosts and ports
sudo tcpdump -i en0 -nn port 4502                         # one AEM port
sudo tcpdump -i en0 -nn host 203.0.113.20 and port 4503   # one publisher
sudo tcpdump -i en0 -nn -c 50 'tcp port 80 or tcp port 443'   # stop after 50 packets
sudo tcpdump -i en0 -nn -w capture.pcap port 4503         # write raw packets to a file (analyse later / in Wireshark)
tcpdump -nn -r capture.pcap                               # read it back, no root needed
sudo tcpdump -i en0 -nn -s 0 -A port 8080                 # print payload as ASCII: see the HTTP request and headers
sudo tcpdump -i en0 -nn -X -c 1 icmp                      # hex + ASCII of one ICMP packet
sudo tcpdump -i en0 -nn -tttt -v port 4502                # full date-time stamps, IP-level detail (TTL, DF, checksum)
sudo tcpdump -i en0 -nn 'tcp[tcpflags] & (tcp-syn|tcp-rst) != 0'   # connection attempts and resets only
sudo tcpdump -i en0 -nn -Q in port 4503                   # inbound only (macOS/Linux)
sudo tcpdump -i en0 -nn -G 300 -w 'aem-%Y%m%d-%H%M.pcap' port 4503    # rotate files every 5 minutes
```

Flags worth memorising:

| Flag | Effect | Why you want it |
|---|---|---|
| `-i en0` | interface; `-i any` on Linux; **macOS: `-i pktap,en0,lo0` or `-i all`** | macOS has no `any`; `pktap` also records which process sent the packet |
| `-nn` | no reverse DNS, no port-name lookup | faster, no stalls, `4502` instead of a guessed service name |
| `-c N` | stop after N packets | a safety net on busy hosts |
| `-s 0` | capture the full packet (default since 4.0 is 262144 bytes, effectively full) | needed for `-A`/`-X` of large payloads on old versions |
| `-w file` / `-r file` | write / read pcap | capture on the server, analyse on your laptop |
| `-A` / `-X` / `-XX` | ASCII / hex+ASCII / hex+ASCII including the Ethernet header | read HTTP, see TLS record types, check Ethernet MACs |
| `-v` `-vv` | more decoding: TTL, DF, IP id, checksum verification | PMTUD and checksum problems |
| `-t` / `-tttt` | no timestamp / full date and time | correlating with AEM logs |
| `-q` | shorter lines | skimming |
| `-e` | print MACs on each line | ARP/switching problems: is the frame going to the right gateway MAC? |
| `-Q in|out` | direction filter | separate what arrived from what you sent |
| `-G secs -w pattern` / `-C MB -W n` | rotate by time / size, keep n files | long-running captures without filling the disk |
| `-k` (macOS only) | print process name and PID per packet (needs `pktap` interface) | "which process is talking to that IP" |
| `-d` | print the compiled filter and exit | check a filter before a long capture (needs `-r` on macOS without root) |

## Filters (BPF): the grammar in one table

The filter is everything after the flags. Keywords combine with `and`, `or`, `not`, and parentheses (quote the whole thing for the shell).

| Want | Filter |
|---|---|
| a host, either direction | `host 203.0.113.20` |
| source / destination only | `src host 203.0.113.20`, `dst port 4503` |
| a network | `net 192.0.2.0/24` |
| a port, either side | `port 4502`; `portrange 4502-4503` |
| a protocol | `tcp`, `udp`, `icmp`, `arp`, `icmp6`, `ip6` |
| not my own SSH session | `not port 22` (always add this when capturing over SSH) |
| TCP flags | `'tcp[tcpflags] & tcp-syn != 0'`, `tcp-rst`, `tcp-fin`, `tcp-push`; handshake only: `'tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn'` |
| HTTP requests by method | `'tcp port 4503 and (tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420)'` (`GET `); easier: `-A` and grep |
| TLS ClientHello only | `'tcp port 443 and tcp[((tcp[12:1] & 0xf0) >> 2)] = 0x16 and tcp[((tcp[12:1] & 0xf0) >> 2)+5] = 0x01'` |
| packets with DF set | `'ip[6] & 0x40 != 0'` |
| ICMP errors only (not echo) | `'icmp and icmp[0] != 8 and icmp[0] != 0'` |
| VLAN-tagged | `vlan` (and then the rest of the filter) |

Rule of thumb: filter as tightly as you can **at capture time** (`-w` with a filter), because the kernel drops what the filter rejects and the host barely notices; filtering a 2 GB file afterwards is slow.

## Reading the one-line format

A TCP handshake, a request and a server reset, read from a file (`-nn -r`):

```
00:13:20.000000 IP 198.51.100.10.52000 > 203.0.113.20.4502: Flags [S], seq 1000, win 65535, length 0
00:13:20.012300 IP 203.0.113.20.4502 > 198.51.100.10.52000: Flags [S.], seq 5000, ack 1001, win 65535, length 0
00:13:20.024600 IP 198.51.100.10.52000 > 203.0.113.20.4502: Flags [.], ack 1, win 65535, length 0
00:13:20.036900 IP 198.51.100.10.52000 > 203.0.113.20.4502: Flags [P.], seq 1:98, ack 1, win 65535, length 97
00:13:20.049200 IP 203.0.113.20.4502 > 198.51.100.10.52000: Flags [P.], seq 1:92, ack 98, win 65535, length 91
00:13:20.061500 IP 203.0.113.20.4502 > 198.51.100.10.52000: Flags [R.], seq 92, ack 98, win 65535, length 0
```

- **Timestamp**, then **`IP`** (or `IP6`, `ARP`), then **`src.port > dst.port`**. With `-nn` the port is a number; without, tcpdump prints a service name from `/etc/services` which is often wrong for AEM ports.
- **`Flags [...]`**: `S` SYN, `.` ACK, `P` PSH, `F` FIN, `R` RST, `U` URG, `W`/`E` ECN. So `[S.]` is SYN-ACK, `[P.]` is data with ACK, `[R.]` is a reset. **Three lines `[S]`, `[S.]`, `[.]` = a completed handshake.** `[S]` repeating with no `[S.]` = nothing is answering (firewall drop or host down). `[S]` answered by `[R.]` = the port is closed (REJECT, or AEM not listening).
- **`seq` and `ack`**: shown **relative** to the handshake once tcpdump has seen the SYN (`seq 1:98` = bytes 1 to 97 of this direction). **If your filter or capture start excludes the SYN, numbers are absolute** (`seq 1001:1098` in the same capture when filtering on PSH only, verified). Add `-S` to always print absolute numbers.
- **`win`**: receive window. A `win 0` from the server with data queued = the application is not reading (an AEM thread pool stall shows up here before it shows up in logs).
- **`length`**: TCP payload bytes. `length 0` lines are pure ACKs.
- **`-v` adds the IP header**: `(tos 0x0, ttl 64, id 4660, offset 0, flags [DF], proto TCP (6), length 40)` and `cksum 0x68f1 (correct)`. **`incorrect` checksums on packets *sent by the capturing host* are normal**: the NIC computes them after capture (checksum offload, see `options=` in [ifconfig.md](ifconfig.md)). On received packets they are real.
- **`-A`** prints the payload after the summary line. For plain HTTP you see `GET /content/... HTTP/1.1`, `Host:`, `Cookie: login-token=...` in clear. For TLS you see only the handshake's SNI hostname in the ClientHello and then noise.
- **`-X`** prints offset, hex, ASCII. The IPv4 header starts at `4500` (version 4, 20-byte header); on Ethernet captures without `-XX` the 14-byte frame header is skipped.

ICMP and ARP lines look different and are read in [icmp.md](icmp.md) and [arp.md](arp.md): `IP a > b: ICMP a unreachable - need to frag (mtu 1400), length 36` is the PMTUD message; `ARP, Request who-has 192.0.2.1 tell 192.0.2.10` / `ARP, Reply 192.0.2.1 is-at 00:00:5e:00:53:01` is the resolution pair.

## Gotchas

- **No root, no capture, and on macOS no filter test either** (use `-d -r file`).
- **Capturing over SSH without `not port 22`** floods the output with your own session and the output's own packets, feeding back forever.
- **Reverse DNS stalls.** Without `-n`, every new IP triggers a PTR lookup that can take seconds and itself appears in the capture. Always `-nn`.
- **"Flags [.]" is not "nothing".** It is an ACK, and a stream of `[.]` with `length 0` from one side while the other sends `[P.]` is normal flow.
- **Wrong interface = empty capture.** Traffic to `localhost` is on `lo0`, not `en0`. On a VPN the traffic is on `utunN`. Dispatcher → publish on the same host never touches the NIC. `tcpdump -D` and `route -n get <dest>` first.
- **macOS has no `-i any`.** Use `-i pktap,en0,lo0` or `-i all`; those captures are written as pcap-ng with process metadata, and are **not promiscuous**.
- **Promiscuous mode shows other hosts' traffic only on a hub, a mirror/SPAN port, or Wi-Fi monitor mode.** On a switched network you see your own traffic plus broadcasts, which is why "tcpdump on my laptop shows nothing from the server" is expected.
- **Offloading lies twice**: wrong checksums on outgoing packets, and **packets larger than the MTU** in captures (`length 65160`) because TSO hands the NIC a giant segment that it splits after capture. Neither is a network fault.
- **Dropped packets** are reported at the end (`N packets dropped by kernel`). A non-zero number means the host could not keep up; tighten the filter or use `-w` without `-v`.
- **Encrypted traffic is opaque.** TLS shows the SNI and record sizes, nothing else. Capture *behind* the TLS terminator (Dispatcher → publish on plain HTTP) or use the application's own logging.

## What this means for AEM

- **AEMaaCS**: you cannot run tcpdump on Adobe's side, and traffic from your side is TLS to the CDN, so a capture tells you only: does the TCP handshake to `cdn.adobeaemcloud.com:443` complete, does the TLS handshake complete, and what SNI you send. That is enough to separate "network/allow list/proxy" from "application" when a browser shows a blank error: `[S]` with no `[S.]` is network; a handshake plus an HTTP 403 inside TLS (which you cannot see here, use `curl -sv`) is the allow list.
- **AMS / on-prem**: the real use. Capture **on the Dispatcher host** towards the publisher: `sudo tcpdump -i eth0 -nn -A port 4503 and not port 22`.
  - **`/renders` misconfiguration**: `[S]` repeating to the wrong IP, or `[R.]` straight back = Dispatcher log's `Unable to connect socket` seen from the wire.
  - **Slow publisher**: the `GET` goes out, the `[.]` ACK comes back at once, and the `HTTP/1.1 200` arrives seconds later = AEM is slow, not the network. If the ACK itself is late or retransmissions (`seq` repeating) appear, it is the network.
  - **PMTUD black hole**: handshake and small requests fine, a large `[P.]` from the publisher never ACKed and retransmitted, no `need to frag` ICMP visible ([icmp.md](icmp.md)).
  - **Which headers really reach AEM** (`X-Forwarded-For`, `Host`, `Via`, the Dispatcher-added `Server-Agent`): `-A` on 4503 shows exactly what Apache forwarded, settling "the CDN strips our header" arguments in a minute.
  - **Replication**: capture `port 4503` on author to see the `POST /bin/receive` and the publisher's response code when the queue says "blocked" with a socket timeout.
  - **Security note**: on plain HTTP you will see Basic-auth replication credentials and `login-token` cookies in `-A` output. Treat capture files as secrets, filter with `-w` and a tight expression, and delete them when done. The same exposure is what ARP poisoning exploits ([arp.md](arp.md)).
- **Local development**: an SDK author on `localhost:4502` lives on `lo0`. `sudo tcpdump -i lo0 -nn -A port 4502` shows the raw requests your frontend or integration test sends, including the cookies and the `CSRF-Token` header, which is often faster than enabling request logging.

## Cheat sheet

- `sudo tcpdump -i <if> -nn [-c N] [-w f.pcap] '<filter>'`; read with `tcpdump -nn -r f.pcap` (no root).
- Always `-nn`, always `not port 22` over SSH, pick the interface with `tcpdump -D` and `route -n get <dest>`.
- `[S]` `[S.]` `[.]` = handshake; `[S]` alone = dropped; `[S]` then `[R.]` = closed; `[P.]` = data; `win 0` = receiver not reading.
- `seq`/`ack` are relative only when the SYN was captured; `-S` for absolute.
- Outgoing bad checksums and over-MTU segments are offloading, not faults.
- `-A` for HTTP in clear, `-v` for TTL/DF/checksum, `-e` for MACs, `-tttt` to match log timestamps.
- macOS: no `-i any` (use `pktap,...`), `-k` shows the process, `-d` needs `-r` without root.
- TLS is opaque; capture on the plain-HTTP hop (Dispatcher → publish) or use application logs.
