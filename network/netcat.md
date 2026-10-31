# netcat (`nc`): TCP/UDP by hand, port checks, listeners, and the flags that differ per build

`nc` (netcat) opens a raw TCP or UDP socket and connects it to your terminal: whatever you type or pipe in goes out on the wire, whatever comes back is printed. With `-l` it listens instead. That is the whole tool, and it is enough for three jobs that come up constantly: **"is this port reachable from here?"**, **"what does this service say if I talk to it directly?"**, and **"what is this client actually sending?"**. Companion docs: [osi-model.md](osi-model.md) (where `nc` sits in the bottom-up test ladder), [udp.md](udp.md) and [icmp.md](icmp.md) (why the UDP mode lies), [tcpdump.md](tcpdump.md) (what to run alongside it).

## The telephone picture

`curl` is a secretary who knows how to talk to a web server. `nc` is the bare telephone: it dials the number (host + port), hands you the handset, and relays sounds in both directions without understanding any of them. That makes it protocol-agnostic (HTTP, SMTP, Redis, a replication agent, anything line-based) and also means it will happily "succeed" at dialling a number nobody answers in a language you expect. A successful `nc` connect proves **layer 4 only**: the TCP handshake completed with *something* on that port. It says nothing about TLS, HTTP, authentication or whether the thing answering is AEM.

## Which `nc` you have

There is no single netcat. The flags differ, which is why copied one-liners fail across machines:

| Build | Where | Tell-tale | Notes |
|---|---|---|---|
| **Apple's `nc`** (BSD-derived) | macOS `/usr/bin/nc` | `nc -h` lists `--apple-*` options, `-G conntimo` | The behaviours below marked "macOS build" apply to this one. |
| **OpenBSD `nc`** | Debian/Ubuntu `netcat-openbsd` (the default `nc` on most modern Linux) | has `-N` (shutdown on stdin EOF), `-q` on Debian's patched build | Closest to Apple's. |
| **GNU / "traditional" netcat** | `netcat-traditional`, old RHEL | `-e prog` (execute on connect), `-q secs` | Avoid `-e` on anything shared; it is a remote shell. |
| **`ncat`** (Nmap project) | `brew install nmap`, `nmap-ncat` on RHEL | `ncat --version`, `--ssl`, `--proxy`, `--sh-exec` | The one to install when you need TLS or predictable timeouts. Waits for the reply after stdin EOF, which the macOS build does not (see gotchas). |

`which -a nc ncat netcat` shows what is installed; `nc -h` shows which dialect it is.

## Client mode

```bash
nc -vz host 443                 # TCP connect, no I/O: "succeeded!" or "Connection refused" / timeout
nc -vz -G 3 host 4503           # same with a 3 s connect timeout (macOS build; see gotchas on -w)
nc -z -G 1 host 4500-4510       # port range, one line per port with -v
nc -4 / -6 host port            # force address family (an AAAA record may route you somewhere else)
nc -s 192.0.2.10 host port      # pick the source address (test a specific interface/VIP)
nc -x proxy.example.net:1080 -X 5 host port    # via SOCKS5; -X connect = HTTP CONNECT proxy
nc -u host 514                  # UDP: each line you type becomes one datagram; nothing confirms delivery
nc -U /path/to/socket           # Unix domain socket
```

Reading the result:

| Output | Meaning |
|---|---|
| `Connection to host port 443 [tcp/https] succeeded!` | Handshake completed. Something listens. Not proof of TLS or HTTP. |
| `connectx to … failed: Connection refused` (macOS) / `Connection refused` | The host is reachable and answered with RST: no listener on that port, or a firewall configured to reject rather than drop. |
| hangs, then `Operation timed out` / nothing | SYN dropped: host down, wrong IP, or a firewall that drops silently. Distinguish from "refused": refused is a fast, positive signal that the network path is fine. |
| `nodename nor servname provided, or not known` | DNS failed before any packet left. Fix with `dig` first ([ip-addresses.md](ip-addresses.md)). |
| `No route to host` | Local routing or ARP problem, not the remote service ([arp.md](arp.md), [ifconfig.md](ifconfig.md)). |

Exit status follows the result (`0` on success, `1` otherwise), so `nc -z -G 3 host 443 && echo up` works in scripts and health checks.

**Talking to a service by hand.** Type the protocol yourself and read the raw reply:

```
$ nc -c host 80
GET /content/site/en.html HTTP/1.1
Host: www.example.net
Connection: close

HTTP/1.1 200 OK
...
```

`-c` sends CRLF line endings, which HTTP and SMTP expect (most servers tolerate bare LF; strict ones do not). This shows exactly which headers come back from a Dispatcher or CDN with no client-side rewriting, and works for any text protocol: `EHLO` to an SMTP relay, `PING` to Redis, `INFO` to a TarMK cold-standby port. For HTTPS use `openssl s_client -connect host:443 -servername host` or `ncat --ssl`; plain `nc` cannot do TLS.

## Listener mode

```bash
nc -l 127.0.0.1 9000            # accept ONE connection on 127.0.0.1:9000, print what arrives, exit when it closes
nc -l -k 9000                   # keep listening after the client disconnects (one client at a time)
nc -l -u 9000                   # UDP listener
nc -l 9000 > upload.bin         # receive a file; on the other side: nc -G 3 host 9000 < upload.bin
```

What it is for:

- **See what a client sends.** Point a replication agent, an HttpClient integration, a webhook or a load balancer health check at `nc -l` and read the request verbatim: the `Host` header, the `Authorization` header, the path, the body encoding. There is no faster way to settle "the agent *is* sending the header" arguments.
- **Prove a firewall path end to end.** `nc -l 4503` on the publisher (with AEM stopped or on a spare port) and `nc -vz dispatcher-side` from the other host: if that fails, nothing in AEM or Apache matters yet.
- **Move a file when `scp` is not available** (jump hosts, containers with no SSH). Unencrypted and unauthenticated; only on a network you trust, and prefer `tar cz | nc` with a checksum (`shasum`) on both ends.

Listening on a port below 1024 needs root. On the macOS build, `nc -l port` without an address binds all interfaces; give `127.0.0.1` explicitly when you only mean local.

## Gotchas

Reproduced on the macOS build unless stated; most carry over to OpenBSD `nc`.

- **`-w` does not bound the connect on macOS.** `nc -z -w 2 203.0.113.1 80` (an unroutable address) returned after **75 s**, the kernel's SYN-retry limit; `-w` only governs the final read. `-G 2` returned in 2.0 s. Use `-G <secs>` for connect timeouts on macOS; on OpenBSD `nc`, `-w` does cover the connect; `ncat -w` covers both.
- **Piped input loses the reply.** `printf 'GET / HTTP/1.0\r\n\r\n' | nc host 80` prints **nothing** on the macOS build: `nc` exits the moment stdin hits EOF, before the server has answered, and `-w` does not change that. Hold stdin open (`(printf '…'; sleep 2) | nc host 80`), type interactively, or use `ncat`, which waits for the reply. OpenBSD `nc` waits too (and `-N` makes it half-close instead of staying open); GNU netcat has `-q <secs>` for the same purpose; `-N` on macOS means something unrelated (probe count).
- **`-z` against your own one-shot listener consumes it.** `nc -l 9000` accepts exactly one connection and exits, so a `nc -vz host 9000` "to check it is up" reports success and the real client that follows gets `Connection refused`. Use `-k`, or skip the probe.
- **UDP `-z` always "succeeds".** `nc -u -z host 514` prints `succeeded!` whenever no ICMP Port Unreachable came back, including for a closed loopback port and for anything behind a firewall that drops. It is not a UDP port check; see [udp.md](udp.md) and [icmp.md](icmp.md). Test the application instead (`dig`, `sntp`, `snmpget`).
- **"succeeded" is not "the service works".** A CDN, load balancer or Dispatcher accepts the TCP connection for every hostname and every path; the 403 from a Cloud Manager IP allow list, a wrong `Host` header, an expired certificate, or a Dispatcher `/filter` deny all come *after* `nc` has already said yes. Continue up the ladder: `openssl s_client`, then `curl -sv` ([osi-model.md](osi-model.md)).
- **Port names come from `/etc/services`.** `[tcp/sd]`, `[udp/discard]` and similar tags are lookups, not detection; `nc` knows nothing about what is behind the port. `-n` suppresses them (and DNS) for speed.
- **Refused versus timeout is the diagnosis.** Refused in milliseconds means the network path is fine and the listener is missing (service down, bound to another address, wrong port). A long silence means a packet is being dropped somewhere, and `tcpdump` on both ends tells you which side ([tcpdump.md](tcpdump.md)).
- **Everything is plaintext.** Whatever you type (Basic auth, `login-token` cookies, replication credentials) travels unencrypted and lands in your shell history. For anything with a secret, use `curl -n`/`--netrc` or `openssl s_client`.
- **`-e` / `--sh-exec` is a backdoor.** Traditional netcat and `ncat` can attach a shell to a port; security scanners flag the binaries for that reason, and some hardened images remove `nc` entirely. Don't leave a listener with it running on a server.
- **IPv6 silently.** If the name has an `AAAA` record, `nc host port` may connect over IPv6 while your allow list or firewall rule is IPv4-only. `-4`/`-6` to be explicit.

## What this means for AEM

- **AEMaaCS:** the only hop you can touch is your client to Adobe's CDN. `nc -vz -G 3 cdn.adobeaemcloud.com 443` (or your custom domain) proves TCP to the Fastly edge, nothing more: the Cloud Manager IP allow list answers with an HTTP 403 *after* a successful handshake, and the origin is unreachable by design, so `nc` cannot help with "author cannot reach my on-prem service" (that is advanced networking, tested from inside AEM with a curl-style servlet or the Cloud Manager egress checks). Where it does help: a developer or CI runner that cannot reach `git.cloudmanager.adobe.com` or `author-pXXXX-eYYYY.adobeaemcloud.com` on 443 at all is a corporate firewall/proxy problem (`-x proxy -X connect` reproduces the proxied path), not an AEM one.
- **AMS / on-prem:** this is the first tool after `ping`. From the Dispatcher host, `nc -vz -G 3 publish-host 4503` separates "firewall or wrong IP in `/renders`" (refused or timeout) from "Apache/AEM config" (succeeded, so read `dispatcher.log` and `error.log` next). From author, `nc -vz publish 4503` before touching a replication agent that reports "connection refused" or "timed out"; from publish, `nc -vz author 4502` for reverse replication; `nc -vz standby 8023` for TarMK cold standby (and `nc -l 8023` on the standby with AEM stopped to prove the path when the firewall team says it is open). `nc -l 4503` on a spare port with the agent pointed at it shows the exact replication POST, headers and Basic auth the agent sends.
- **Local SDK / 6.5 development:** `nc -vz 127.0.0.1 4502` tells you whether the JVM has opened its HTTP port yet (useful in start scripts: loop until it succeeds, then run the readiness curl), and `nc -c localhost 4502` lets you send a raw request with hand-written headers (`CSRF-Token`, `Referer`, `Cookie`) to see which one Sling actually requires.
- **It never replaces `curl -u` or a browser.** A green `nc` only closes the "is there a network path" question; the AEM answer (login, Dispatcher filters, CDN cache state, 403 from allow lists) lives one layer up.

## Cheat sheet

- `nc -vz [-G secs] host port` = TCP connect test. `succeeded!` = handshake only; `refused` = reachable, no listener; silence = dropped.
- macOS build: **`-G` for connect timeout, `-w` is read timeout only** (an unroutable host hangs ~75 s with `-w`). OpenBSD `nc`: `-w` covers connect. `ncat -w` covers both.
- macOS build **exits on stdin EOF without reading the reply**: `printf … | nc` prints nothing. Hold stdin open, type interactively, or use `ncat`.
- `-c` = CRLF line endings (HTTP/SMTP by hand). `-n` = no DNS/service-name lookups. `-4`/`-6` = address family. `-s` = source address. `-x host:port -X 5|connect` = proxy. `-u` = UDP. `-U` = Unix socket. `lo-hi` = port range.
- `nc -l [addr] port` = one connection then exit; `-k` to keep listening. Below 1024 needs root. A `-z` probe eats a one-shot listener.
- `nc -u -z` is not a UDP port test; silence is ambiguous (udp.md, icmp.md).
- No TLS: use `openssl s_client -servername` or `ncat --ssl`. Everything typed is plaintext and in your history.
- Builds differ: Apple/OpenBSD (`-G`/`-N`/`-k`), GNU traditional (`-q`, `-e`), `ncat` (`--ssl`, `--proxy`, `--sh-exec`). `nc -h` tells you which.
- AEM: on AEMaaCS proves only client→CDN TCP (allow-list 403 comes later); on AMS/on-prem the first test for Dispatcher→4503, replication 4502/4503, cold standby 8023, and `nc -l` to read what an agent really sends.
