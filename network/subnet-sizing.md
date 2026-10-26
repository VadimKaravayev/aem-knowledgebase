# Subnet sizing: how many addresses a prefix holds, how many hosts can use them, and how many devices a network really supports

Three different questions hide behind "how many devices fit on this network?": how many **addresses** a prefix contains (pure arithmetic), how many of them a **host can use** (addresses minus the ones the protocol reserves), and how many **devices the network tolerates** before DHCP, broadcast traffic or the Wi-Fi radio give out, which is usually the smaller number. Addresses in the examples use the documentation ranges. Companion docs: [ip-addresses.md](ip-addresses.md) (what an address and CIDR notation mean), [dhcp.md](dhcp.md) (where the pool limit comes from), [arp.md](arp.md) (why broadcast domains have a practical ceiling).

## The arithmetic: 2 to the power of the host bits

An IPv4 address is 32 bits. The prefix length after the slash says how many of them name the **network**; the rest name the **host** inside it.

```
host bits   = 32 − prefix length
addresses   = 2 ^ host bits
usable hosts = addresses − 2        (network address and broadcast address)
```

Worked example, `192.0.2.0/26`:

```
host bits    = 32 − 26 = 6
addresses    = 2^6 = 64          → 192.0.2.0 … 192.0.2.63
usable hosts = 64 − 2 = 62       → 192.0.2.1 … 192.0.2.62
network      = 192.0.2.0          (all host bits 0)
broadcast    = 192.0.2.63         (all host bits 1)
```

The **subnet mask** is the same information written as dotted bits: `/26` = `255.255.255.192`, because 26 one-bits is `11111111.11111111.11111111.11000000` and `11000000` = 192. macOS `ifconfig` prints it in hex, `0xffffffc0` ([ifconfig.md](ifconfig.md)).

The table everyone ends up memorising:

| Prefix | Mask | Addresses | Usable hosts | Typical use |
|---|---|---|---|---|
| /32 | 255.255.255.255 | 1 | 1 (host route) | One machine in an allow list |
| /31 | 255.255.255.254 | 2 | 2 (RFC 3021) | Point-to-point link between two routers |
| /30 | 255.255.255.252 | 4 | 2 | Legacy point-to-point link |
| /29 | 255.255.255.248 | 8 | 6 | A tiny DMZ, a handful of servers |
| /28 | 255.255.255.240 | 16 | 14 | Small server segment |
| /27 | 255.255.255.224 | 32 | 30 | |
| /26 | 255.255.255.192 | 64 | 62 | |
| /25 | 255.255.255.128 | 128 | 126 | |
| /24 | 255.255.255.0 | 256 | 254 | The default home or office LAN |
| /23 | 255.255.254.0 | 512 | 510 | |
| /22 | 255.255.252.0 | 1 024 | 1 022 | A floor or a large office Wi-Fi |
| /21 | 255.255.248.0 | 2 048 | 2 046 | |
| /20 | 255.255.240.0 | 4 096 | 4 094 | A typical cloud subnet |
| /16 | 255.255.0.0 | 65 536 | 65 534 | A whole site or VPC |
| /8 | 255.0.0.0 | 16 777 216 | 16 777 214 | `10.0.0.0/8`, the whole private class A |

Each step down the prefix **doubles** the count. Going from `/24` to `/23` does not add one address, it adds another 256.

## Why minus two, and when it is not minus two

- The **network address** (host bits all zero, `192.0.2.0` in a `/24`) identifies the subnet itself in routing tables and cannot be assigned to a host.
- The **broadcast address** (host bits all one, `192.0.2.255`) reaches every host on the subnet; ARP requests and DHCP Discover go there ([arp.md](arp.md), [dhcp.md](dhcp.md)).
- **/31 is the exception**: RFC 3021 allows both addresses on a two-address link because a point-to-point link needs no broadcast. Routers use it; most desktop stacks accept it; older gear does not.
- **/32** is a single host, used for loopbacks, host routes and allow-list entries. It has no network or broadcast of its own.

Then subtract what *you* reserve, which the formula does not know about:

- The **default gateway** takes one usable address (`192.0.2.1` by convention), so a `/24` has **253** for devices, not 254.
- Anything static: printers, access points, management interfaces, the DHCP server itself, a VRRP/HSRP virtual gateway (one more address per redundant router pair).
- **Cloud providers reserve more.** AWS takes the first four and the last address of every subnet (network, router, DNS, future use, broadcast), so a `/24` gives **251** and the smallest allowed subnet, `/28`, gives **11**. Azure reserves five as well. The arithmetic says 14 for a `/28`; the console will show 11.

## How many devices a network supports

The usable-host count is the ceiling; the real limit comes earlier, from one of these.

**The DHCP pool.** Devices get addresses from the pool the server hands out, which is almost always smaller than the subnet: a consumer router often defaults to a pool of 100 or 150 inside a `/24`, and some leave only `x.x.x.100–199`. The 101st device gets no lease and self-assigns `169.254.x.x` ([dhcp.md](dhcp.md)). Lease time matters too: with a 24-hour lease, a device that visited this morning still holds its address tonight, so a busy guest network needs a pool sized for everyone seen within a lease, not everyone present. Short leases (1–2 hours) stretch a pool.

**The broadcast domain.** Every ARP request, DHCP Discover, mDNS announcement and Windows/SMB browse packet reaches every host. The cost scales with the number of hosts, and every host must process every broadcast in software. Rule of thumb for wired networks: a few hundred hosts per VLAN is comfortable, `/22` (about 1 000 hosts) is a sensible upper bound, and anything larger is split into VLANs with a router between them. A `/16` flat LAN is valid arithmetic and a terrible network.

**The Wi-Fi radio.** Addresses are not the constraint on Wi-Fi; airtime is. A single consumer access point serves roughly 30–50 active clients well regardless of how large its subnet is, and enterprise access points are rated per radio (typically 50–100 recommended, several hundred associated). A `/24` behind one home router runs out of radio long before it runs out of the 253 addresses.

**The hardware tables.** Consumer routers cap concurrent NAT sessions and ARP/DHCP table sizes in the low thousands; cheap switches cap MAC table entries. These bite on networks with many IoT devices before the subnet does.

**The router's own licence or spec.** Some firewalls and SD-WAN boxes are licensed per concurrent IP; the subnet may be `/22` but the box stops at 250.

Putting it together for the common question "how many devices can my home network support": a `/24` offers 253 addresses, the router's DHCP pool offers 100–250 of them, the Wi-Fi radio copes with 30–50 active clients per access point, and the router's NAT table tolerates a couple of thousand sessions. The answer is "the smallest of those", and it is normally the radio.

## Sizing a subnet: work backwards

1. Count the devices expected at peak, **plus** gateway, static infrastructure and cloud-reserved addresses.
2. Add growth: double it, because the next size up costs nothing in private space.
3. Pick the smallest prefix whose usable count covers that. Devices needed 300 → 2^9 = 512 → `/23`.
4. Check the practical limits above: if the result is bigger than about `/22`, split by VLAN instead.
5. Make sure the prefix aligns: a `/23` must start on an even third octet (`192.0.2.0/23` covers `.2.0–.3.255`; `192.0.3.0/23` is invalid).

Reverse direction, given a count, the shortcut is `ceil(log2(devices + 2))` host bits: 50 devices → 52 → 6 bits → `/26`; 1 000 devices → 1 002 → 10 bits → `/22`.

Tools that do the arithmetic: `ipcalc 192.0.2.0/26` (Linux, `brew install ipcalc` on macOS), `sipcalc`, Python `ipaddress`:

```
python3 -c "import ipaddress; n=ipaddress.ip_network('192.0.2.0/26'); print(n.num_addresses, n.num_addresses-2, n.broadcast_address)"
```

## IPv6: the question stops making sense

An IPv6 address is 128 bits and a LAN is a `/64` by convention, which is 2^64 ≈ 1.8 × 10^19 addresses per segment; there is no broadcast address and no "minus two". SLAAC hands every device an address from the interface identifier, so the pool never runs out. The only sizing decision in IPv6 is how many `/64` segments a site gets (`/48` = 65 536 of them, `/56` = 256, the usual home allocation). The device ceiling is then entirely the broadcast-domain, radio and table limits from the previous section, which do not change with address size.

## Gotchas

- **Class A/B/C is dead.** Classful sizes (`/8`, `/16`, `/24`) are historical; CIDR allows any prefix. A `/24` is "class C" only in old documentation and exam questions.
- **`/24` is not "256 devices".** It is 254 by the protocol, 253 after the gateway, 251 on AWS, and 100 if the DHCP pool says so.
- **Mask typos are silent.** A host configured `/24` on a `/25` network can talk to half the subnet and not the other half, with no error; symptoms look like a flaky switch. `ipcalc` both ends when "some hosts can't reach some hosts".
- **Overlapping subnets break routing.** Two sites both using `192.168.1.0/24` cannot be joined by VPN without NAT; it is the single most common reason to choose an unusual range (`10.137.42.0/24`) for a new office or lab.
- **Supernetting an allow list.** Giving a partner `/24` when their egress is one address grants 255 strangers access; allow lists take `/32` per address, and the prefix in a list is a *range*, not a count of hosts ([ip-addresses.md](ip-addresses.md)).
- **Broadcast storms scale with hosts squared** in the worst case (every host answering every broadcast); the practical VLAN ceiling exists for this, not for address exhaustion.

## What this means for AEM

- **AMS and on-prem segments are small by design.** An author, two publishers, two Dispatchers and a load balancer fit in a `/28`; what matters is leaving room for a cold standby, a second author for upgrades, and the VRRP address, so `/27` is the usual request. A `/24` per tier is not wasteful in private space and avoids re-addressing during a scale-out.
- **Cloud VPC sizing for AEM.** Pick the VPC `/16` and per-tier subnets `/24` or `/23` across two or three availability zones, remembering the provider's five reserved addresses per subnet; an autoscaling publish tier plus Dispatchers plus load balancer interfaces (which also consume addresses) exhausts a `/28` quickly. See [[aem-on-aws-whitepaper]].
- **Cloud Manager and Dispatcher IP allow lists** are written as CIDR, and the prefix is a range grant: an office with a single NAT address is `/32`; a corporate egress block is whatever the network team states, never a rounded-up `/24`. Count the addresses the prefix covers before saving it.
- **Load tests from one subnet.** A `/24` of load generators is 253 source addresses; if the Dispatcher or CDN rate-limits per source IP, that is the number you are dividing the target rate by.
- **"How many editors can use author?"** is not a subnet question. Author connections are HTTP sessions through a load balancer; the limiting resource is JVM threads and the repository, not addresses ([[ssh-tunneling]] for reaching it, not sizing it).

## Cheat sheet

- `addresses = 2^(32 − prefix)`, `usable = addresses − 2` (network + broadcast); `/31` keeps both, `/32` is one host.
- Each prefix step doubles: `/24` 254, `/23` 510, `/22` 1 022, `/20` 4 094, `/16` 65 534.
- Subtract gateway and statics; AWS and Azure reserve 5 per subnet (`/24` → 251, `/28` → 11).
- Devices supported = min(usable hosts, DHCP pool, broadcast-domain comfort ≈ a few hundred to 1 000, Wi-Fi ≈ 30–50 active per consumer AP, router table limits).
- Sizing: host bits = `ceil(log2(devices + 2))`, double for growth, split by VLAN above about `/22`, align the start address.
- IPv6: `/64` per LAN, effectively unlimited addresses, no minus two; sizing is about counting `/64`s.
- AEM: tiers fit in `/27`–`/24`; cloud subnets lose five addresses each and load-balancer interfaces eat more; allow-list prefixes are range grants, `/32` for a single office egress.
