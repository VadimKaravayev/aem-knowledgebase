# ARP: resolving an IP address to a MAC address

ARP (Address Resolution Protocol, RFC 826) answers one question: "which hardware address do I put on the frame for IP address X?" **It works only within a single local network** (one subnet, one broadcast domain): requests are broadcast frames, so they stop at the first router, and a host can resolve only addresses on its own segment. For any other destination it resolves the default gateway instead and lets the router take over. Every IPv4 packet that leaves a host goes through it, which makes it the first thing to check when "ping works but the connection hangs". It also has no authentication, which is covered in its own section below. Addresses in the examples use the documentation ranges (`192.0.2.0/24`, MAC prefix `00:00:5e:00:53`); the macOS values are the kernel defaults of a current release. Companion docs: [mac-addresses.md](mac-addresses.md) (what a MAC is, stale ARP after failover, gratuitous ARP), [ip-addresses.md](ip-addresses.md), [icmp.md](icmp.md) (Neighbor Discovery, ARP's IPv6 replacement, rides on ICMPv6).

## The room picture

IP addresses are names. MAC addresses are faces. A network card delivers frames to faces, not names. So before a laptop can send anything to `192.0.2.1` it shouts into the room: "who is 192.0.2.1?" The device with that name answers "me, and this is my face". The laptop remembers the pairing for a while so it does not have to shout every time.

Two consequences follow:

- **Only the room hears the shout.** ARP never crosses a router. For anything off-subnet the host ARPs for the **default gateway** and hands the packet to the router's face; the router repeats the game on the next segment. A server, CDN or licence server on the internet never sees your MAC and never receives your ARP.
- **Anyone in the room can answer.** There is no "prove you are 192.0.2.1". The protocol trusts the first (or the latest) reply. This is the gap that [ARP poisoning](#arp-poisoning) exploits.

## The protocol in four lines

1. **Request** (broadcast, destination MAC `ff:ff:ff:ff:ff:ff`): "Who has 192.0.2.1? Tell 192.0.2.10." Every host on the segment receives it; only the owner answers.
2. **Reply** (unicast to the asker): "192.0.2.1 is at 00:00:5e:00:53:01."
3. **Cache**: the asker stores the pair in its ARP table (neighbour cache) with a timer.
4. **Send**: the IP packet goes out in a frame addressed to that MAC.

ARP is its own EtherType (`0x0806`), beside IPv4 (`0x0800`). It is **not** carried inside IP, which is why a host with no IP configured yet (DHCP discovery, APIPA probing) can still send it, and why IP-layer firewalls cannot filter it.

Two variants matter operationally:

- **Gratuitous ARP**: an unsolicited announcement "192.0.2.1 is at …" that a host sends when it comes up or takes over an IP (VRRP/keepalived, cloud re-attach). Neighbours update their caches without asking. This is how failover is supposed to work, and it is exactly the mechanism poisoning abuses.
- **ARP probe**: "who has 192.0.2.10?" sent with a sender IP of `0.0.0.0` to check nobody already owns the address before claiming it (RFC 5227, duplicate address detection). macOS exposes this as `send_conflicting_probes: 1` below.

## What the cache looks like

```
$ arp -a
? (192.0.2.1)   at 00:00:5e:00:53:01 on en0 ifscope [ethernet]      # router (factory OUI)
? (192.0.2.20)  at 2:0:5e:0:53:2a    on en0 ifscope [ethernet]      # randomised MAC (x2/x6/xA/xE second digit)
? (192.0.2.255) at ff:ff:ff:ff:ff:ff on en0 ifscope [ethernet]      # subnet broadcast
mdns.mcast.net (224.0.0.251) at 1:0:5e:0:0:fb on en0 ifscope permanent [ethernet]   # multicast: derived, never ARPed
```

Reading notes for macOS:

- macOS prints hex pairs **without leading zeros** (`2:0:5e:0:53:2a`), which breaks copy-paste into tools expecting two digits.
- `permanent` entries are multicast mappings computed from the IP (`01:00:5e:` + low 23 bits), not learned. A `permanent` entry for a *unicast* IP means someone ran `arp -s`.
- The `?` is a failed reverse DNS lookup, not an unknown host.

macOS kernel timers (defaults, all under `sysctl net.link.ether.inet`):

| Tunable | Value | Meaning |
|---|---|---|
| `max_age` | 1200 s | an entry is considered stale after **20 minutes** and gets re-resolved |
| `prune_intvl` | 300 s | the cache is swept for expired entries every 5 minutes |
| `arp_llreach_base` | 120 s | an entry is treated as "reachable" for 2 minutes after traffic confirms it |
| `maxtries` | 5 | requests sent before a resolution is given up |
| `arp_unicast_lim` | 3 | refresh attempts sent unicast to the known MAC before falling back to broadcast |
| `host_down_time` | 20 s | how long a host is treated as down after resolution fails |
| `send_conflicting_probes` | 1 | duplicate-address probing enabled |
| `proxyall` | 0 | the host does not answer ARP on behalf of others (proxy ARP off) |
| `log_arp_warnings` | 0 | **conflicts and MAC changes are not logged by default**; set to 1 to see them in the system log |

Linux defaults for comparison (from kernel docs): `base_reachable_time_ms` 30 s, `gc_stale_time` 60 s, `arp_accept` 0 (an unsolicited reply **updates an existing entry** but does not create a new one). Windows randomises reachable time around 15–45 s. The practical point: **every mainstream OS re-resolves within a minute or two of silence, and every one of them accepts a reply that overwrites what it already has**. That is why stale entries clear themselves after a failover, and also why poisoning works.

Useful commands:

```
arp -a                                  # dump the cache (all OSes)
arp -d 192.0.2.1                        # drop one entry (sudo) when a stale one is suspected
sudo arp -s 192.0.2.1 00:00:5e:00:53:01     # static entry: survives until reboot or arp -d; 'temp' makes it age
ip neigh show                           # Linux equivalent of arp -a (STALE/REACHABLE/FAILED states visible)
sudo tcpdump -i en0 -n arp              # watch requests, replies, and gratuitous announcements live
```

## ARP as a diagnostic

`arp -a` on a host tells you which MAC the box is *actually* sending to for the gateway, the Dispatcher, or the other publish node. When "ping works but the connection hangs" or "one Dispatcher cannot reach one publisher", compare the MAC on both sides before touching application configuration. A mismatch is a network problem, not an application one.

The classic outage is **stale ARP after failover**: an IP moves between hosts (VRRP, keepalived, a manual re-IP), the new owner has a different MAC, and any neighbour that did not hear the gratuitous ARP keeps sending to the old face until its entry ages out. The symptom is "new server is up but unreachable for a few minutes from one host"; the fix is `arp -d` on the stuck host, and the prevention is making sure the takeover sends gratuitous ARP. Details in [mac-addresses.md](mac-addresses.md).

## ARP poisoning

ARP poisoning (also called ARP spoofing) is the abuse of the "anyone in the room can answer" property: a host on the segment sends unsolicited replies claiming someone else's IP, gets inserted between two parties, and becomes the cheapest man-in-the-middle on a LAN. The attacker must already be on the segment: shared Wi-Fi, an office LAN, or a compromised machine inside it. It cannot be done from the internet.

### The lie

Target: laptop `192.0.2.10`, gateway `192.0.2.1`, attacker MAC `aa:aa:aa:aa:aa:aa`.

1. Attacker sends the laptop an unsolicited reply (or gratuitous ARP): "192.0.2.1 is at aa:aa:…". The laptop overwrites its gateway entry.
2. Attacker sends the gateway: "192.0.2.10 is at aa:aa:…". The gateway overwrites its entry for the laptop.
3. Both now frame their traffic for the other party to the attacker's MAC. The attacker has IP forwarding on and relays every packet, so the connection keeps working and nothing looks broken.
4. The attacker repeats the lies every few seconds, because the victims would otherwise re-resolve (see timers above) and the real owner would win the race back.

In the room picture: someone shouts "I'm Alex!" every time anyone asks for Alex, then quietly passes the messages along so nobody notices they went through the wrong hands.

What the attacker gets from the middle:

- **Read** anything not end-to-end encrypted: plain HTTP, legacy POP/IMAP/FTP/SMTP, internal tools without TLS, **AEM author/publish on 4502/4503 over plain HTTP** inside a LAN, including Basic-auth headers and `login-token` cookies.
- **Modify** traffic in flight: inject content into pages, swap a download, alter a replication payload.
- **Drop** traffic: the victim is silently offline (denial of service) while `arp -a` looks fine.
- **Attack TLS** from the middle: only by presenting a fake certificate. A client that validates certificates gets a warning or a hard failure (HSTS); a client that was taught to click through, or a service with `-k`/`insecure` in its config, is fully exposed.

Related tricks that use the same trust gap: **DHCP spoofing** (hand out yourself as the gateway, no ARP needed), **IPv6 Router Advertisement spoofing** (same idea on networks that "don't use IPv6" but have it enabled by default), and **ARP flooding** of a switch's MAC table so it falls back to hub behaviour and leaks all traffic to every port.

### Spotting it

- **Two IPs, one MAC.** The gateway and some other host sharing a MAC in `arp -a` is the signature. Multicast and the broadcast entry do not count.
- **The gateway's MAC changed.** Compare against a known-good value (the router's label, or a previous dump). On macOS set `sysctl net.link.ether.inet.log_arp_warnings=1` to log changes; on Linux `arpwatch` mails on every new or flipped pairing.
- **Everything is slightly slower and the first hop in `traceroute` is wrong**: the attacker's host appears as hop 1 before the real gateway, or the gateway RTT doubles.
- **Certificate warnings on sites that never had them.** Treat these as hostile on shared networks, not as "the Wi-Fi is weird".
- A packet capture shows **ARP replies nobody asked for**, arriving periodically, from one MAC.

### Defending against it

| Layer | Control | What it does | Limits |
|---|---|---|---|
| Switch | **Dynamic ARP Inspection (DAI)** + DHCP snooping | the switch builds an IP↔MAC↔port table from DHCP leases and **drops any ARP reply that contradicts it** | managed switches only; static-IP hosts need manual bindings; useless on an unmanaged switch or a consumer-grade access point |
| Switch / AP | **Client isolation / private VLAN** on guest and BYOD Wi-Fi | clients can reach the gateway but not each other, so there is no "room" to shout into | defeats peer-to-peer features (AirPlay, printers) |
| Host | **Static ARP for the gateway** (`arp -s`, Linux `ip neigh add … nud permanent`) | the entry cannot be overwritten by a reply | breaks when the router is replaced; practical for servers and Dispatcher hosts, not laptops |
| Host | **End-to-end encryption**: TLS with validated certificates, SSH, VPN | the attacker still gets the bytes but cannot read or alter them; a VPN also hides destinations | TLS protects only if clients refuse bad certificates; mutual TLS or pinning for machine-to-machine |
| Network | **Segmentation**: servers in their own VLAN, management interfaces on a separate one | a poisoned office LAN cannot reach the server segment's ARP domain at all | needs routing/firewall between segments |
| Monitoring | `arpwatch`, IDS rules (Suricata/Zeek `arp` logs), switch port-security alerts | detect flips and duplicates; the only control that tells you it *happened* | detection, not prevention |

The order that matters: **encrypt everything** (makes the attack unprofitable), **isolate untrusted clients** (removes the room), **DAI where the switch supports it** (drops the lie at the port). Static ARP and host-based detection are for the handful of boxes you really care about.

### Failover looks like poisoning and vice versa

An IP that moves between publishers legitimately changes its MAC and announces it with gratuitous ARP, which is byte-for-byte what an attacker sends. Before calling it an attack, check whether a failover just happened; before calling it a failover, check whether the new MAC belongs to a server you own.

## What this means for AEM

- **AEMaaCS**: nothing on your side touches ARP. Author and publish live in Adobe's Kubernetes network; the only ARP domain you own is the office LAN your browser sits on. The exposure there is a poisoned client reading your session, which TLS to `*.adobeaemcloud.com` already defeats unless the user accepts a bad certificate. Cloud Manager IP allow lists do not help against a man-in-the-middle who is on your own network; they see your office's public IP either way.
- **AMS / on-prem**: the realistic attack surface is **internal plain-HTTP hops**: Dispatcher → publish on 4503, author → publish replication on 4503, load balancer → Dispatcher, author on 4502 reached from an office LAN. A poisoned host on that segment sees replication payloads, Basic-auth replication credentials, and author `login-token` cookies in clear. Mitigations in order: terminate TLS on publish (or put a TLS-terminating proxy in front of 4503 and point replication agents at it), put the AEM tier in its own VLAN with no end-user clients, and set static ARP for the gateway on Dispatcher and publish hosts if the switch cannot do DAI.
- **Stale ARP after a publisher failover** is the ARP problem you will actually meet: "Dispatcher lost one publisher for a few minutes after the IP moved". Check `arp -a` on the Dispatcher host against the publisher's real MAC before touching Dispatcher or OSGi configuration.

## One-paragraph summary

ARP maps an IP to a MAC by broadcasting a question on the local segment and caching the answer for a few minutes (20 min max age on macOS, about a minute on Linux). It rides its own EtherType, never crosses a router, and is announced unsolicited (gratuitous ARP) when an IP moves, which is how failover works and why a stale entry can leave a new host unreachable from one neighbour until the timer expires. It has no authentication, so a host on the same segment can send unsolicited replies claiming the gateway's IP, be inserted between a victim and the router, and read, alter or drop anything not end-to-end encrypted. Spot poisoning by two IPs sharing one MAC or a gateway MAC that changed; prevent it with Dynamic ARP Inspection, client isolation, static entries on critical hosts, and TLS everywhere. For AEM the only places it bites are plain-HTTP internal hops on AMS/on-prem (Dispatcher→publish, replication, author on 4502), which is one more reason to terminate TLS inside the data centre and keep the AEM tier on its own VLAN.
