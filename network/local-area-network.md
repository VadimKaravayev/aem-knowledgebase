# Local Area Network: the broadcast domain, what stays inside it, switches versus routers, VLANs, and how to tell what is "local"

A LAN is the set of devices that can reach each other **without a router**: one Ethernet/Wi-Fi segment, one IP subnet, one broadcast domain. Everything that makes a LAN feel different from "the internet" follows from that one property: broadcasts (ARP, DHCP, mDNS) reach every member, MAC addresses are visible, the switch forwards by MAC, nothing is translated, and the only way out is the default gateway. Companion docs: [ip-addresses.md](ip-addresses.md) (private ranges, NAT), [subnet-sizing.md](subnet-sizing.md) (how big a LAN can be), [arp.md](arp.md), [mac-addresses.md](mac-addresses.md), [dhcp.md](dhcp.md), [ifconfig.md](ifconfig.md), [osi-model.md](osi-model.md).

## The one-room picture

A LAN is one room. Anyone in the room can shout ("who has 192.0.2.1?") and everyone hears it; anyone can hand a note directly to anyone else by name (MAC address). The **switch** is the room's floor: it learns where each person stands and passes notes straight to them. The **router** is the door: to reach another room you hand the note to the doorkeeper (default gateway), who rebuilds the envelope and carries it on. Shouting never passes through the door. Wi-Fi is the same room with the walls removed: the access point is the floor, and everyone's notes are on the air, which is why encryption (WPA2/3) and **client isolation** exist.

Consequences that explain most LAN behaviour:

- **Same subnet = direct.** If the destination IP is inside your interface's network/mask, the OS ARPs for *its* MAC and sends straight to it. If not, it ARPs for the **gateway's** MAC and sends there. A wrong netmask makes half the room "another room" (unreachable) or another room "this room" (ARP for a host that will never answer). [arp.md](arp.md), [ifconfig.md](ifconfig.md).
- **Broadcasts stop at the router.** ARP, DHCP Discover, NetBIOS, mDNS/Bonjour (`.local` names, AirPlay, printers), SSDP/UPnP, Wake-on-LAN: all work inside the room and none cross the door unless a relay or proxy is configured ([dhcp.md](dhcp.md) relay, mDNS reflector). "I can ping it by IP but not find it by name.local" across a VPN or VLAN is this.
- **MACs are visible only here.** The switch, the AP and every neighbour see your MAC; nothing beyond the router does ([mac-addresses.md](mac-addresses.md)).
- **No NAT inside.** Inside the room everyone sees everyone's real address and port. NAT happens at the door, once, on the way to the internet: outside hosts see one public IP for the whole room ([ip-addresses.md](ip-addresses.md)).
- **The LAN is a trust boundary whether you meant it or not.** Anyone in the room can ARP-poison, sniff Wi-Fi, or hit every unauthenticated port. "Only reachable on the LAN" is a weaker control than it sounds.

## Switch, router, access point, and the boxes that are all three

| Device | Layer | Forwards by | Boundary it makes |
|---|---|---|---|
| **Hub** (obsolete) | 1 | repeats every bit to every port | none: one collision domain |
| **Switch** | 2 | MAC address table (learned from source MACs) | one broadcast domain per VLAN; separates collision domains |
| **Access point** | 2 | bridges Wi-Fi to the wired segment | same room as the wired LAN unless it isolates clients or uses its own VLAN |
| **Router** | 3 | IP routing table; rewrites MACs and TTL per hop | separates broadcast domains; NAT if it also does that |
| **Consumer "router"** | 1–3 + 7 | switch + AP + router + NAT + DHCP + DNS forwarder in one box | the whole home: everything on it is one LAN unless "guest network" splits it |
| **L3 switch** | 2 + 3 | switches within a VLAN, routes between VLANs | office core: many LANs in one chassis |

The consumer box is why the terms blur: its four Ethernet ports and the Wi-Fi are one switch, so wired and wireless devices share one broadcast domain and one DHCP pool. The "guest network" is a second VLAN with isolation, which is also why guest devices cannot see the printer.

## VLANs: many rooms on one floor

802.1Q adds a 4-byte tag (12-bit VLAN ID, 1–4094) to each Ethernet frame so one physical switch carries several independent broadcast domains. **Access ports** belong to one VLAN and strip tags toward the device; **trunk ports** carry many VLANs tagged (between switches, to a hypervisor, to an AP serving several SSIDs). Each VLAN is its own subnet with its own gateway; crossing between them is routing, usually with a firewall policy, and that is the point: office / servers / management / guest / IoT are rooms that only meet at a controlled door. Things to know:

- A device on an access port never sees the tag; "put it on VLAN 20" is a switch-side setting. A VM or container host *does* see tags on a trunk and must be told which VLAN its virtual interface uses.
- The **native/untagged VLAN** on a trunk carries frames with no tag; mismatched native VLANs on the two ends silently merge two rooms.
- Broadcast storms and misbehaving devices are contained per VLAN; one noisy IoT gadget cannot flood the server VLAN.
- Wi-Fi: one AP, several SSIDs, each mapped to a VLAN; the "guest" SSID is normally VLAN + client isolation + rate limit.

## Addressing a LAN

- Private ranges by convention: `192.168.x.0/24` at home, `10.x.y.0/24` (or bigger) in offices and clouds, `172.16–31.x.0` in Docker and some corporate nets. Collisions between two LANs with the same range (home `192.168.0.0/24` and a client VPN that also uses it) break routing: the OS prefers the directly connected route and the VPN side becomes unreachable ([ip-addresses.md](ip-addresses.md)).
- Size with the prefix: `/24` = 254 usable; a bigger room means more broadcast noise and a bigger blast radius, so offices split by function rather than grow one `/16` ([subnet-sizing.md](subnet-sizing.md)).
- Gateway by convention `.1` or `.254`; DHCP hands out the rest; reservations pin known devices (keyed on MAC, which randomised Wi-Fi MACs defeat, [dhcp.md](dhcp.md)).
- `169.254.x.x` on an interface = DHCP never answered; the device is in the room but nobody gave it a number, so it can talk only to other link-local hosts.
- IPv6 on a LAN: every interface has a link-local `fe80::` address whether or not IPv6 is routed; neighbours are found with ICMPv6 Neighbour Discovery (multicast, not broadcast), so blocking ICMPv6 on a LAN breaks IPv6 entirely ([icmp.md](icmp.md)).

## Is X on my LAN? Reading the routing table

```
$ netstat -rn -f inet          # macOS/BSD; Linux: ip route
default        192.0.2.1   UGScg   en0      ← everything not matched below goes to the gateway
192.0.2        link#14     UCS     en0      ← the LAN: directly connected, no gateway
192.0.2.1      0:0:5e:0:53:1  UHLWIir en0   ← a resolved neighbour (its MAC, from ARP)
169.254        link#14     UCS     en0      ← link-local, also "this room"
```

`link#N` as the gateway means "on this interface, no router": that destination is local. Anything matched by `default` leaves through the door. `route -n get <ip>` (macOS) or `ip route get <ip>` (Linux) answers the question for one address, including which interface and whether a VPN (`utunN`) has claimed it.

## Seeing who is in the room

```bash
arp -a                                     # neighbours this host has talked to (macOS drops leading zeros in MACs)
ifconfig en0 | grep -E "inet |ether"       # my address, mask, broadcast, current MAC (randomised on Wi-Fi by default)
ping -c 1 192.0.2.255                      # broadcast ping: macOS and many devices answer, Linux ignores it (icmp_echo_ignore_broadcasts=1)
dns-sd -B _services._dns-sd._udp local.    # macOS: browse Bonjour services (printers, AirPlay, _http._tcp dev servers)
avahi-browse -a                            # Linux equivalent
nmap -sn 192.0.2.0/24                      # ping/ARP sweep of your own LAN (only on networks you administer)
sudo tcpdump -i en0 -nn 'arp or (udp port 67 or 68) or udp port 5353'   # the LAN chatter: ARP, DHCP, mDNS
```

Broadcast ping and `nmap -sn` populate the ARP table, so `arp -a` afterwards lists everyone who answered. Devices with a firewall that drops ICMP still show in ARP if they replied to the ARP probe `nmap` sends on a local segment.

## Wi-Fi specifics

- Shared medium: everyone's frames are on the air; WPA2/3 encrypts per client with a key derived from the password, so on a **pre-shared-key** network anyone who knows the password and captured your handshake can decrypt your traffic (WPA3 fixes this with SAE). Enterprise (802.1X) gives per-user keys.
- **Client isolation** (AP/guest setting) blocks station-to-station frames: devices get internet but cannot see each other. Explains "my phone cannot reach my laptop's dev server on the office Wi-Fi".
- **Band steering / separate 2.4 and 5 GHz SSIDs** are usually still one LAN; separate SSIDs with separate VLANs are not.
- **MAC randomisation** ([mac-addresses.md](mac-addresses.md)) is on by default on current phones and laptops: a new MAC per network and sometimes per day, so MAC-based reservations, allow lists and captive-portal memory break.
- Wi-Fi is half-duplex with retransmissions; "LAN is slow" on Wi-Fi is usually interference or a far client dragging the airtime, not the switch.

## Gotchas

- **Wrong mask = half the room missing.** `/23` on one host, `/24` on the rest: it can reach some neighbours directly and ARPs in vain for others. First thing to compare in `ifconfig` output on both sides.
- **Two LANs, one range.** Home `192.168.0.0/24` and a client VPN pushing the same range: whichever route is more specific or connected wins, and the other becomes silently unreachable. Fix by renumbering one side; this is why corporate LANs avoid `192.168.0.0` and `192.168.1.0`.
- **Guest network / client isolation** looks like a firewall on the target. Test from a wired port or another SSID before blaming the server.
- **"It works by IP, not by name.local."** mDNS does not cross routers or VPNs; use DNS names or an mDNS reflector.
- **DHCP from the wrong room.** A rogue DHCP server (a plugged-in consumer router with its LAN port facing the office switch) hands out its own addresses and gateway to part of the LAN; symptoms are random hosts on a `192.168.1.x` island. `ipconfig getpacket en0` (macOS) shows which server answered.
- **Broadcast storms** from a loop (two cables between two switches without STP) take the whole VLAN down; the lights on every port blink in unison.
- **The LAN is not private because it is "local".** Unauthenticated admin ports (4502 on a dev AEM, Redis, a database, Jenkins) on an office LAN are reachable by every laptop, guest and IoT device in that VLAN.
- **Link-local `169.254.x.x` is a symptom, not a setting.** No DHCP answer; check cable/SSID, VLAN assignment, DHCP pool exhaustion ([dhcp.md](dhcp.md)).

## What this means for AEM

- **AEMaaCS:** nothing about your LAN reaches Adobe; what matters is the **public IP of your LAN's NAT** (the office egress address), because that is what Cloud Manager IP allow lists and the WAF see. Everyone behind one office router shares one allow-list entry; home workers each have their own, split-tunnel VPNs change which one applies, and IPv6 egress bypasses an IPv4-only list ([ip-addresses.md](ip-addresses.md)). Author/publish/Dispatcher on AEMaaCS live in Adobe's network; no LAN of yours contains them.
- **AMS / on-prem:** AEM is deployed as several LANs (VLANs): a DMZ for Dispatcher/web tier, an app VLAN for author and publish, a management VLAN for monitoring and backups, often a separate one for the data store and database. Dispatcher → publish on 4503, author → publish replication on 4503, publish → author reverse replication on 4502, TarMK cold standby on 8023, and JMX/JDWP all cross VLAN boundaries, so each is a firewall rule, not "same LAN, just works". When one Dispatcher cannot reach one publisher, confirm first whether they are in the same VLAN (`netstat -rn`: `link#` or gateway) because the diagnosis differs: same room → ARP/mask/host firewall ([arp.md](arp.md)); different room → routing/firewall policy ([netcat.md](netcat.md) to test the port). Replication and Dispatcher traffic inside a trusted app VLAN is commonly plain HTTP; that is acceptable only if the VLAN really is isolated, since every host in it can read the replication credentials and `login-token` cookies ([tcpdump.md](tcpdump.md)).
- **Local SDK / 6.5 development:** the quickstart listens on all interfaces, so `http://<your LAN IP>:4502` works from a phone or a colleague's laptop in the same room, which is the cheap way to test responsive layouts on real devices. Three things break it: Wi-Fi client isolation (use a wired port or a hotspot), the macOS application firewall prompt for `java`, and absolute URLs that AEM generates with `localhost` (Sling mapping, Core Components' canonical links, Dispatcher-less `Externalizer` config) so the page loads but assets and links point back at the phone itself. Use the LAN IP or a `.local`/hosts-file name consistently, and remember that `nslookup` cannot resolve either ([nslookup.md](nslookup.md)). The same openness means a dev author with default `admin`/`admin` is reachable by everyone on the office LAN; bind it to `127.0.0.1` (Felix `org.apache.felix.http` host property) or change the password.
- **AEM Screens, kiosks and digital signage:** players sit on a store or office LAN and reach publish through the door (NAT, proxy, allow list); content download problems are LAN egress and DNS problems first, and a player on a guest/isolated VLAN cannot be reached by its device manager or peers.
- **Office LAN → cloud author access** is a VPN or zero-trust question, not a LAN one: once the packet leaves the room, only the egress IP and the identity (IMS) matter.

## Cheat sheet

- LAN = one subnet = one broadcast domain = reachable without a router; the gateway is the only door. Same subnet → ARP for the host; other subnet → ARP for the gateway.
- Broadcasts (ARP, DHCP, mDNS/Bonjour, SSDP, WoL) never cross the router; MACs are visible only inside; NAT happens once at the door.
- Switch = L2, MAC table, splits collision domains; router = L3, splits broadcast domains; consumer box = switch + AP + router + NAT + DHCP + DNS forwarder; L3 switch = many VLANs routed in one chassis.
- VLAN = 802.1Q tag (1–4094), one broadcast domain and subnet each; access port untagged, trunk tagged, mind the native VLAN; guest Wi-Fi = VLAN + client isolation.
- `netstat -rn` / `ip route`: `link#N` = local, `default via` = through the door; `route -n get <ip>` / `ip route get <ip>` for one address.
- See the room: `arp -a`, broadcast ping (Linux ignores it), `dns-sd -B` / `avahi-browse`, `nmap -sn <your subnet>`, `tcpdump 'arp or udp port 67 or 68 or udp port 5353'`.
- Gotchas: mismatched masks, two LANs with the same range (VPN), client isolation, rogue DHCP, switch loops, `169.254` = no DHCP, "local" ≠ private.
- AEM: AEMaaCS sees only your NAT egress IP (allow lists); AMS is several VLANs with firewall rules on 4502/4503/8023; local SDK is reachable at `<LAN IP>:4502` from any device in the room unless isolated, so lock the admin password or bind to loopback.
