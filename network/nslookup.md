# nslookup: asking a DNS server directly, reading the answer, and why it disagrees with your browser

`nslookup` sends one DNS query to one server and prints the reply. That is all it does, and the two halves of that sentence are the source of every surprise: **one server** (it asks the resolver you name or the first one in `/etc/resolv.conf`, not the resolver your applications use), and **one query** (it does not follow the OS's search list, scoped resolvers, `/etc/hosts`, or Happy Eyeballs). Companion docs: [ip-addresses.md](ip-addresses.md) (A vs CNAME vs apex, the phone-book picture), [udp.md](udp.md) (DNS is UDP 53 with TCP fallback), [netcat.md](netcat.md) and [osi-model.md](osi-model.md) (the bottom-up ladder: `nslookup` is the L7-name step before `nc`), [cloud-manager-custom-domain-names.md](../cloud-service/cloud-manager-custom-domain-names.md) (the records it is usually checking).

## The phone-book picture

DNS is a chain of phone books. Your application asks the OS; the OS asks a **recursive resolver** (your router, the corporate DNS, `1.1.1.1`); the resolver either answers from its **cache** or walks the **authoritative** servers for the zone and caches the result for the record's **TTL**. `nslookup` lets you ring any book in that chain yourself and read the page as printed, which is how you find out *which* book is stale, wrong, or returning something different from the others.

## What it actually does

- Reads the nameserver from `/etc/resolv.conf` (macOS, Linux) unless you pass one as the second argument: `nslookup name server`.
- Sends a UDP query on port 53 (`-vc` forces TCP; `-port=` changes the port), waits `-timeout=` seconds, retries `-retry=` times.
- Prints the server it used, then the answer section. It **hides TTLs and the DNS header** unless you add `-debug`.
- Exits **1 on NXDOMAIN or no server reachable, 0 on everything else, including "No answer"** (name exists, no record of that type). Scripts that test `$?` miss the NODATA case; grep the output instead.
- Interactive mode (`nslookup` with no arguments) accepts `server 192.0.2.53`, `set type=cname`, `set debug`, then names, one per line; the same options exist as `-type=`, `-debug` on the command line.
- Version on macOS and most Linux is BIND's (`nslookup -version` → `9.x`). Windows ships its own with different habits (below).

## Reading the output

```
$ nslookup www.example.net
Server:         192.0.2.53
Address:        192.0.2.53#53

Non-authoritative answer:
www.example.net canonical name = cdn.adobeaemcloud.com.
cdn.adobeaemcloud.com   canonical name = adobe-aem.map.fastly.net.
Name:   adobe-aem.map.fastly.net
Address: 151.101.3.10
```

| Line | Meaning |
|---|---|
| `Server:` / `Address: …#53` | Which resolver answered. **Check this first**: a VPN, a captive portal or a stale `resolv.conf` means you are asking a different book than you think. |
| `Non-authoritative answer:` | Came from a recursive resolver's cache or recursion, not from the zone's own nameserver. The record may be up to one TTL old. Absent when you query an authoritative server directly. |
| `canonical name = …` | A CNAME hop. `nslookup` follows the chain and prints every hop, then the final `Name:`/`Address:` pairs, which belong to the **last** name, not the one you asked for. |
| `Name:` / `Address:` repeated | Several A/AAAA records; clients pick one (round-robin or Happy Eyeballs). |
| `Authoritative answers can be found from:` followed by nothing | Normal filler on BSD/macOS builds; not an error. |
| `*** Can't find X: No answer` | **NODATA**: the name exists but has no record of the requested type. Typically "you asked for A and it only has a CNAME to something that resolves elsewhere" or "no AAAA". Exit code 0. |
| `** server can't find X: NXDOMAIN` | The name does not exist in that resolver's view. Exit code 1. Check the search list (`example` vs `example.corp.example.net`) and typos before blaming the zone. |
| `;; connection timed out; no servers could be reached` | UDP 53 to that server is blocked or the server is down; try `-vc` (TCP) and another server before concluding anything. |
| `REFUSED` | The server exists but will not answer that query for you: usually a resolver that refuses non-recursive queries (`-norecurse`) or queries from outside its network. |

## Recipes

```bash
nslookup www.example.net                    # A/AAAA via the default resolver (CNAME chain shown)
nslookup -type=cname www.example.net        # just the alias target: is it cdn.adobeaemcloud.com yet?
nslookup -type=a example.net                # apex: must be A records (no CNAME at the apex)
nslookup -type=aaaa www.example.net         # IPv6 presence: an IPv4-only allow list treats these clients differently
nslookup -type=txt _aemverification.www.example.net   # the Cloud Manager ownership TXT record
nslookup -type=ns example.net               # who is authoritative; then ask them directly:
nslookup www.example.net ns1.example.net    # bypass every cache; no "Non-authoritative" line
nslookup www.example.net 1.1.1.1            # a public resolver: is the record visible outside the corporate network?
nslookup -type=soa example.net              # serial (did the zone change?) and minimum (negative-cache TTL)
nslookup -type=mx example.net               # mail: relevant for AEM outbound SMTP routing
nslookup 203.0.113.10                       # reverse (PTR) lookup; many CDN/cloud IPs have none
nslookup -debug www.example.net             # full packet: flags, TTL per record, authority/additional sections
nslookup -vc www.example.net                # TCP instead of UDP: works but UDP does not → UDP 53 is filtered
nslookup -timeout=2 -retry=1 name server    # fail fast in scripts
```

**Cutover check.** Before flipping a domain, query (1) the authoritative servers, (2) your corporate resolver, (3) a public resolver, and compare. Differences are either propagation (the old record's TTL has not expired in that cache; `-debug` shows the remaining `ttl`) or **GeoDNS/anycast** (the CDN hands out different addresses per region; normal, not a fault). Lower the TTL a day *before* the change so the caches drain fast afterwards.

## `nslookup` versus the resolver your app uses

This is the gotcha that costs the most time: **`nslookup` succeeding or failing says nothing certain about what Java, `curl` or the browser will get**, because they go through the OS resolver and `nslookup` does not.

- **macOS**: applications use `mDNSResponder` with the per-interface, per-domain resolvers that `scutil --dns` lists (VPN split DNS, `*.corp` scoped resolvers, `/etc/hosts`). `nslookup`, `dig` and `host` read only `/etc/resolv.conf`, i.e. resolver #1. So "works in the browser over VPN but `nslookup` says NXDOMAIN" (or the reverse) is expected. Ask the real path with `dscacheutil -q host -a name www.example.net` or `dns-sd -G v4v6 www.example.net`; flush with `sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder`.
- **Linux**: with `systemd-resolved`, `/etc/resolv.conf` usually points at the stub `127.0.0.53`, so `nslookup` does go through the same resolver as apps, but still skips `/etc/hosts` and `nsswitch.conf`; `resolvectl query name` and `getent hosts name` show the application view. Without `resolved`, `/etc/hosts` is the usual discrepancy.
- **Java/AEM**: the JVM caches successful lookups for `networkaddress.cache.ttl` (30 s by default; indefinite when a security manager is installed) and failures for `networkaddress.cache.negative.ttl` (10 s by default). After a DNS fix, `nslookup` is right immediately and AEM may keep throwing `UnknownHostException` until the cache expires or the JVM restarts.
- **Windows**: `nslookup` uses the DNS servers of the first adapter only, does a reverse lookup of that server first (`Default Server: UnKnown` is cosmetic), ignores the hosts file and the DNS client cache; `Resolve-DnsName` and `ipconfig /displaydns` show the client's view, `ipconfig /flushdns` clears it.

## `nslookup`, `dig`, `host`

All three ship on macOS and most Linux. `nslookup` is the one that is also on Windows and the one most people know, so it is what tickets quote. `dig +short name` is better for scripts (one value per line, TTL with `dig +noall +answer`, `+trace` to walk the delegation from the root, `+tcp`, `+bufsize=`, exit codes that follow the rcode). `host -t cname name` is the terse middle. For anything beyond "what is the A record", reach for `dig`; this doc exists because `nslookup` is what you will be handed.

## Gotchas

- **Which server answered is the first thing to read.** VPN clients, captive portals, Docker and DHCP all rewrite `resolv.conf`.
- **"Non-authoritative" is not "wrong".** It means cached. If it disagrees with the authoritative server, the difference is TTL or geo, and a cache cannot be flushed from outside; wait or query another resolver.
- **The addresses printed belong to the end of the CNAME chain.** A CNAME to a CDN resolves to the CDN's anycast addresses, which differ by region and over time. Do not hard-code them in allow lists or `/etc/hosts`.
- **NODATA exits 0, NXDOMAIN exits 1.** `nslookup name >/dev/null && …` passes a name with no A record.
- **Round-robin output order rotates.** Two runs printing the addresses in a different order is not a change.
- **No TTL unless `-debug`.** Everyone who asks "has it propagated yet" needs the TTL; use `-debug` or `dig`.
- **The search list hides typos and reveals internals.** A bare `author` may resolve via `search corp.example.net` on the office network and fail everywhere else; use fully qualified names (trailing dot: `author.corp.example.net.`) in configs.
- **Public resolvers may be blocked.** `nslookup name 1.1.1.1` timing out on a corporate network usually means outbound UDP 53 is restricted to the internal resolvers, not that the record is missing.
- **`nslookup` does not do DNSSEC validation or EDNS tricks.** A record that validates incorrectly can be visible in `nslookup` against the authoritative server and still `SERVFAIL` through a validating resolver; `dig +dnssec` shows why.
- **Reverse lookups are decorative.** Most CDN, cloud and AEM-related addresses have no PTR; absence proves nothing, and a PTR is not proof of ownership.

## What this means for AEM

- **AEMaaCS custom domains.** The checks Cloud Manager's "Verify" performs are the ones to run yourself first: `nslookup -type=cname www.example.net` must end in `cdn.adobeaemcloud.com` → `adobe-aem.map.fastly.net`; an apex `nslookup -type=a example.net` must return Adobe's four A records (a CNAME at the apex is invalid, hence ALIAS/ANAME or A records); `nslookup -type=txt _aemverification.www.example.net` must return the ownership token. Query the authoritative servers, not your laptop's resolver, when Verify fails and your own lookup "looks right"; and remember the TXT record routes nothing, the CNAME routes everything ([cloud-manager-custom-domain-names.md](../cloud-service/cloud-manager-custom-domain-names.md)).
- **"Works for me, 403/404 for the client."** Different resolvers → different CNAME targets during a migration (old origin vs Adobe's CDN); compare `nslookup name <their resolver>` with yours before touching Dispatcher or allow lists. Adobe's edge addresses are anycast and differ per region; a client seeing a different `151.101.x.x` than you is normal.
- **AEMaaCS outbound and advanced networking.** DNS for AEM's own outbound calls is resolved inside Adobe's infrastructure, not through your VPN or port forwards; a name that resolves only on your corporate DNS (split-horizon) will not resolve from AEMaaCS. Publish the name publicly or use the IP via a port forward.
- **AMS / on-prem.** `UnknownHostException` in `error.log` for replication, LDAP, SMTP or integration hosts: run `nslookup host` *on the AEM server* (its resolver, its search list), then `nslookup host <authoritative>`; if the host resolves but AEM still fails, it is the JVM cache (`networkaddress.cache.ttl` in `java.security`, or restart). Replication agents and Dispatcher `/renders` entries should use fully qualified names or IPs; a short name that depends on the search list breaks on the one host whose DHCP lease was renewed differently.
- **Local SDK / 6.5 dev.** `nslookup localhost` is not how `localhost` resolves (that is `/etc/hosts`); a vhost mapping test like `nslookup site.local` fails by design while the browser works. Use `dscacheutil`/`getent` for hosts-file names.
- **Before `nc`, after nothing.** In the "site is down" ladder, name resolution is step one: if `nslookup` and the application resolver agree on an address, move to `nc -vz host 443`, `openssl s_client`, `curl -sv` ([osi-model.md](osi-model.md)).

## Cheat sheet

- `nslookup name [server]`; `-type=a|aaaa|cname|ns|soa|txt|mx|ptr|any`; `-debug` for TTL/flags; `-vc` TCP; `-port=`, `-timeout=`, `-retry=`; interactive `server …`, `set type=…`.
- First line to read is `Server:`; "Non-authoritative" = cached (TTL-old at worst); no such line when asking an authoritative NS directly.
- CNAME chains are printed hop by hop; the final addresses belong to the last name (CDN anycast, varies by region).
- Exit 1 only for NXDOMAIN/unreachable; **NODATA ("No answer") exits 0**.
- It bypasses the OS resolver: no `/etc/hosts`, no macOS scoped/VPN resolvers (`scutil --dns`, `dscacheutil -q host -a name …`), no Windows DNS client cache (`Resolve-DnsName`), no JVM cache (`networkaddress.cache.ttl`).
- Cutover: compare authoritative vs corporate vs public resolver; lower TTL beforehand; `dig +short`/`+trace` for scripts and delegation walks.
- AEM: verify `cname → cdn.adobeaemcloud.com → adobe-aem.map.fastly.net`, apex A records and `_aemverification` TXT against the authoritative servers; AEMaaCS resolves its outbound names itself (split-horizon names are invisible); on AMS run it on the AEM host and remember the JVM cache.
