# DHCP: how a host gets its address, the DORA exchange, leases, and relays

DHCP (Dynamic Host Configuration Protocol, RFC 2131) hands a host its IP address, subnet mask, default gateway, DNS servers and a handful of other options the moment it joins a network, so nobody types those by hand. It runs over UDP, server port 67, client port 68, and it is the reason a laptop works on any Wi-Fi without configuration. Addresses in the examples use the documentation ranges. Companion docs: [ip-addresses.md](ip-addresses.md) (what the address it hands out means), [mac-addresses.md](mac-addresses.md) (the identity it keys on), [arp.md](arp.md) (what happens right after the lease is granted), [ifconfig.md](ifconfig.md) (where the result shows up).

## The hotel-desk picture

A host arriving on a network is a guest walking into a hotel with no room. It cannot phone reception because it has no room number yet, so it shouts into the lobby: "any desk, I need a room". Every desk that hears shouts back an offer: "room 203.0.113.57, checkout in 24 hours, the exit is that way, the concierge is over there". The guest picks one, announces the choice to the whole lobby so the other desks release their offers, and the chosen desk confirms. The room is **lent**, not given: the guest must come back before checkout to extend it or lose it.

Three things follow:

- **The shout only fills one lobby.** DHCP discovery is a broadcast, so it stops at the router. Other subnets need a **relay** at the desk that forwards the shout to a central server.
- **Any desk can answer.** There is no authentication. A second server on the segment, deliberate or accidental, races the real one.
- **The desk remembers faces, not names.** The server keys everything on the client's MAC address or client identifier. Change the face and you are a new guest.

## The exchange: DORA

| Step | Who | Sent to | Carries |
|---|---|---|---|
| **Discover** | client | broadcast, from `0.0.0.0` to `255.255.255.255` | client MAC, optional hostname and "parameter request list" of options it wants |
| **Offer** | server | broadcast or unicast to the client MAC | proposed address (`yiaddr`), lease time, mask, router, DNS, server identifier |
| **Request** | client | broadcast | the accepted address and the chosen server's identifier, so other servers withdraw |
| **Acknowledge** | server | broadcast or unicast | the final lease and options; the client may now use the address |

A server can also answer **NAK** (refused), which sends the client back to Discover. The whole exchange is usually four packets in well under a second; a Discover that gets no Offer is retried with backoff, and the client falls back to a self-assigned **link-local** `169.254.0.0/16` address, which is the signature of "no DHCP server reachable".

Before using the offered address the client usually sends a gratuitous ARP for it ([arp.md](arp.md)); if something answers, the address is in use and the client declines it.

## Leases, renewal and rebinding

| Point in the lease | What the client does |
|---|---|
| **T1 = 50 %** | Sends a unicast Request to the server that granted the lease; a normal renewal, invisible to the user |
| **T2 = 87.5 %** | Server did not answer; broadcasts a Request so **any** server can rebind the lease |
| **100 %** | Still nothing; the client **drops the address** and starts over with Discover |

A **Release** message hands the address back early; the "renew DHCP lease" button on macOS does a release and a fresh Discover. Lease lengths are a trade-off: short leases (hours) recycle addresses on busy guest networks, long leases (days) keep server and client addresses stable; a server that goes down matters only when leases start expiring, not immediately.

## Options: more than an address

DHCP carries typed **options**. The ones that decide whether the network "works":

- **1** subnet mask, **3** router (default gateway), **6** DNS servers, **51** lease time, **54** server identifier.
- **15** domain name, **119** domain search list, **42** NTP servers, **26** interface MTU ([mtu.md](mtu.md)).
- **121** classless static routes (overrides option 3 when present; the usual way a VPN or split-tunnel pushes routes).
- **43/60** vendor class, **66/67** TFTP server and boot file for PXE boot, **82** relay agent information.

The practical consequence: a wrong DNS or gateway option on the server produces "I have an address but nothing works" or "pings work but names don't resolve" on every client at once.

## Reservations, static addresses and the MAC

The server identifies a client by its **hardware address** (and optionally a client identifier). A **reservation** (static lease) maps one MAC to one fixed address; it is how servers, printers and management interfaces get a stable address without touching their own configuration. Two things break it:

- **Randomised Wi-Fi MACs** on recent macOS, iOS, Android and Windows present a per-network random address, so the reservation never matches and the device lands in the dynamic pool ([mac-addresses.md](mac-addresses.md)).
- **Replacing the card** or moving a VM to a new virtual NIC changes the MAC; the old reservation stays orphaned and the host comes up on a different address.

A truly **static** address configured on the host itself bypasses DHCP entirely and must be excluded from the server's pool, or the server will eventually offer it to someone else and the two will collide.

## Across subnets: the relay

Broadcasts die at the router, so each subnet either needs its own server or a **DHCP relay agent** (`ip helper-address` on Cisco gear, `dhcrelay`, `isc-dhcp-relay`) on its router. The relay takes the broadcast Discover, puts its own interface address in the `giaddr` field, and unicasts it to the central server; the server uses `giaddr` to pick the right pool and sends the reply back through the relay. This is how one server serves a whole campus and why "DHCP works on VLAN 10 but not VLAN 20" is almost always a missing or wrong helper address on the VLAN 20 interface.

## Seeing it

```
ipconfig getpacket en0                  # macOS: the last ACK in full: yiaddr, lease time, router, DNS, every option
ipconfig getoption en0 lease_time       # one option
sudo ipconfig set en0 DHCP              # release and re-request
networksetup -getinfo "Wi-Fi"           # address, mask, router as the OS sees them
```

```
cat /var/lib/dhcp/dhclient.leases       # Linux, dhclient
cat /run/systemd/netif/leases/*         # Linux, systemd-networkd
sudo dhclient -v eth0                   # watch DORA happen
journalctl -u NetworkManager | grep -i dhcp
```

To watch it on the wire ([tcpdump.md](tcpdump.md)):

```
sudo tcpdump -i en0 -nn -v 'udp port 67 or udp port 68'
```

The `-v` is what prints the message type (Discover/Offer/Request/ACK) and the options.

## Gotchas

- **Rogue DHCP server.** A consumer router plugged in LAN-side, a VM with a bridged DHCP server, or a Windows ICS share hands out its own gateway to anyone who asks first. Symptom: some hosts on the segment get the wrong gateway or a `192.168.x.x` address on a `10.x` network. Find it with the tcpdump above; the Offer's server identifier is the culprit. Managed switches prevent it with **DHCP snooping** (only trusted ports may send Offers).
- **Pool exhaustion.** New clients get no Offer and self-assign `169.254.x.x` while existing clients keep working, so it looks like "only new devices are broken". Short leases or a bigger pool.
- **Two servers with overlapping pools.** Each hands out addresses from the same range; collisions appear as intermittent "duplicate IP" and sessions that drop when the other host ARPs.
- **DHCP sets DNS.** A server pushing a dead or internal-only DNS server is the most common "connected but nothing loads". `ipconfig getpacket` shows exactly what was pushed.
- **VPN clients and option 121.** Split-tunnel routes arrive as classless static routes; a client that ignores option 121 ends up sending everything through the tunnel or nothing.
- **DHCPv6 is a different protocol.** IPv6 hosts usually self-configure with SLAAC (router advertisements) and use DHCPv6 (UDP 546/547) only for extra options such as DNS; "DHCP works, v6 doesn't" is a router-advertisement question, not a DHCP one.
- **Cloud instances.** AWS, Azure and GCP hand out addresses via their own DHCP on the virtual network; the lease is tied to the instance's network interface, not negotiable, and the "DHCP options set" on the VPC is where DNS and domain name come from.

## What this means for AEM

- **Server side, almost nothing.** AEM author, publish and Dispatcher hosts on AMS or on-prem run on reserved or static addresses; AEMaaCS pods are addressed by the platform. Replication agents, Dispatcher `renders`, and cold-standby configuration all point at hostnames or fixed IPs, and a server whose address changed because a reservation broke is the one case where DHCP causes an AEM outage: replication queues blocked, Dispatcher returning 502 for a host it can no longer reach.
- **The laptop reaching author.** When the AEM console will not load on an office or VPN network, the DHCP-pushed DNS server and routes are the first suspect: `ipconfig getpacket en0` shows what you were given, and a DNS server that cannot resolve the author hostname or a missing option-121 route to the AMS subnet explains it before any AEM diagnostics.
- **Local development.** A Docker or VM running AEM on a bridged network takes a DHCP lease from the office network; the address changes between sessions, so point hosts files and Dispatcher configs at a reservation or at a host-only network, not at the last address you saw.

## Cheat sheet

- DHCP = UDP 67 (server) / 68 (client), RFC 2131. Hands out address, mask, gateway, DNS, lease time and other options.
- **DORA**: Discover (broadcast) → Offer → Request (broadcast, picks one) → Acknowledge. NAK sends the client back to Discover. No server = `169.254.x.x`.
- Lease: renew unicast at **50 %**, rebind broadcast at **87.5 %**, drop at **100 %**.
- Keyed on the **MAC**: reservations break when the MAC changes or is randomised.
- Broadcast stops at the router: other subnets need a **relay** (`giaddr`, helper address).
- Options that decide if the network works: 1 mask, 3 router, 6 DNS, 51 lease, 121 static routes.
- Inspect: macOS `ipconfig getpacket en0`; Linux lease files or `dhclient -v`; wire `tcpdump -v 'udp port 67 or 68'`.
- Gotchas: rogue server (DHCP snooping fixes), pool exhaustion, overlapping pools, bad DNS option, DHCPv6 ≠ DHCP.
- AEM: servers are static; only a broken reservation or a laptop's DHCP-pushed DNS/routes matter.
