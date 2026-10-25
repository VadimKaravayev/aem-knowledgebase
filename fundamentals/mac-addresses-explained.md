# MAC addresses: the mailbox name on the local link, and the other MAC (message authentication code)

"MAC" means two unrelated things in this knowledgebase. In networking it is the **MAC address** (Media Access Control), the hardware identifier of a network interface. In security it is a **Message Authentication Code**, a keyed integrity tag. This doc is mostly about the first and ends with the second so a search for either lands here. Verified on macOS on 2026-10-25 on a home Wi-Fi LAN. Companion docs: [ip-addresses-explained.md](ip-addresses-explained.md), [icmp-explained.md](../network/icmp-explained.md), [internet-checksum.md](internet-checksum.md).

## The apartment-building picture

An IP address is the building's street address, which the postal system routes by. The MAC address is the **name on the mailbox in the lobby**. The truck only needs the street address; the last step, lobby to mailbox, needs the name. Across the internet nobody cares about your mailbox name. On your own floor it is the only thing that matters.

## What it looks like

48 bits, written as six hex pairs: `6c:4c:bc:12:34:56`.

- **First three pairs = OUI** (Organizationally Unique Identifier), assigned to the manufacturer by the IEEE. A lookup on the prefix usually names the vendor.
- **Last three pairs** = the device's number within that vendor.
- **Two flag bits in the first byte:** lowest bit set = multicast/group address; second-lowest bit set = **locally administered** (software-assigned, not factory). So a first byte of `x2`, `x6`, `xA` or `xE` means "this MAC was made up by an OS or hypervisor".
- `ff:ff:ff:ff:ff:ff` = broadcast to everyone on the segment.
- macOS `arp -a` prints pairs without leading zeros (`ce:92:e:ac:…`), which trips up copy-paste into tools that expect two digits.

## Where it lives and where it stops

Every Ethernet or Wi-Fi frame carries a source and destination MAC. **Switches** learn which MAC sits on which port and forward by that table. **Routers strip the frame and build a new one** for the next link, so the MAC changes at every hop. The server at the far end never sees your MAC, only that of the last router before it. **A MAC address never leaves the local network**, which is why web analytics, licence servers and CDNs cannot see it, and why "block their MAC" is meaningless against anything beyond your own router.

Layering in one line: Ethernet frame (MAC → MAC) carries an IP packet (IP → IP) which carries TCP/UDP/ICMP. The MTU limit in [mtu-explained.md](mtu-explained.md) is the IP packet; the 14-byte Ethernet header with the two MACs sits outside it.

## How IP finds a MAC: ARP and Neighbor Discovery

To send to an IP on its own subnet, a host broadcasts an **ARP** (Address Resolution Protocol) request: "who has 192.168.0.1?" The owner replies with its MAC and the asker caches it for a few minutes. For an off-subnet destination the host ARPs for the **default gateway** instead and hands the packet to the router. IPv6 does the same job with **ICMPv6 Neighbor Discovery** (Neighbor Solicitation/Advertisement, multicast instead of broadcast), which is why a firewall that blocks ICMPv6 kills IPv6 entirely.

```bash
ifconfig en0 | grep ether        # your MAC on macOS (ip link show on Linux)
arp -a                           # ARP cache: which MAC answers for each local IP
arp -d 192.168.0.1               # drop one entry (sudo) when a stale one is suspected
```

Verified ARP cache on a home LAN:

```
? (192.168.0.1)   at 6c:4c:bc:…  on en0   ← router: factory OUI
? (192.168.0.123) at ce:92:e:…   on en0   ← phone: ce = locally administered
? (192.168.0.146) at 12:1b:f5:…  on en0   ← laptop: 12 = locally administered
? (192.168.0.172) at 66:b3:fc:…  on en0   ← 66 = locally administered
```

This machine's own Wi-Fi MAC began with `0a`, also locally administered. **Every client on the LAN except the router was using a randomised MAC.** That is the state of a normal 2026 network and the reason for the first gotcha below.

## Gotchas

- **Not actually fixed.** Any OS can override it. iOS 14+, Android 10+, Windows 10+ (opt-in) and macOS Sequoia+ use a **random MAC per Wi-Fi network** by default; macOS can also rotate it. "Allow only these MACs" on a Wi-Fi router, DHCP reservations keyed by MAC, captive-portal remembering, or a licence tied to a MAC all break in ways that look random, often after an OS update.
- **Not a security control.** Spoofing is one command (`ifconfig en0 ether …` on macOS, `ip link set dev eth0 address …` on Linux). MAC filtering keeps out accidents, not attackers.
- **Unique only per segment.** Duplicates on different networks never meet. **Duplicates on the same switch** make the switch's table flap and traffic reach the wrong host intermittently; cloned VMs with copied MACs are the usual cause. Symptom: a host reachable one second, not the next, with nothing in its own logs.
- **Stale ARP after failover.** When an IP moves to a new host (VRRP/keepalived, a manual swap, a cloud re-attach), neighbours keep sending to the old MAC until their ARP entry expires or a **gratuitous ARP** from the new owner updates them. Looks like "the new server is up but unreachable for exactly N minutes, then fine".
- **Virtual MACs everywhere.** VMs, containers, bonds, VRRP virtual routers (`00:00:5e:00:01:xx`) and Docker bridges (`02:42:…`) all have synthetic MACs; the locally-administered bit marks them.
- **Wi-Fi frames carry three or four MACs** (sender, receiver, access point, sometimes the far end of a bridge) because the access point relays. Ethernet carries two.
- **MAC ≠ identity across subnets.** Logs on a server in another subnet show the gateway's MAC for every client. Only IP (and above) identify the client there.

## What this means for AEM

Almost nothing at the application layer. Where it appears:

- **AMS / on-prem:** a Dispatcher that loses one publish instance for a few minutes after a failover or re-IP, then recovers by itself, is a stale ARP entry (or a missing gratuitous ARP) until proven otherwise. Clear the entry on the Dispatcher host or fix the failover tool to announce. Duplicate MACs from cloned publish VMs give the intermittent variant.
- **Licensing:** AEM 6.x licence files are bound to the licence key and customer name, **not** to a MAC, so cloning or re-hosting does not invalidate them. (Some other Adobe server products of that era were MAC-bound; AEM was not.)
- **AEMaaCS:** nothing. Adobe owns every link, and Cloud Manager IP allow lists are IP-based at the CDN.
- **Wi-Fi MAC randomisation and your office allow list:** if an "office network only" rule for author access is implemented at the Wi-Fi layer by MAC, staff drop off it when their devices rotate; use the office's public IP in the Cloud Manager allow list instead.

## The other MAC: Message Authentication Code

A short tag computed from a message **and a secret key**, so a receiver holding the same key can verify the message was not altered and came from a key holder. It is what a checksum ([internet-checksum.md](internet-checksum.md)) is not: without the key an attacker cannot recompute it after changing the data.

- **HMAC** (hash-based MAC, e.g. HMAC-SHA256) is the common construction; `HS256` JWTs, signed cookies and most webhook signatures (Adobe I/O Events, GitHub, Stripe) use it. In modern TLS the MAC is folded into the AEAD cipher (AES-GCM, ChaCha20-Poly1305) rather than computed separately, but plays the same role.
- **AEM touches this constantly:** the **encapsulated token** that lets a login survive across publish instances without sticky sessions is an HMAC over the token payload. Its key (the `hmac` file, alongside `master`, created on first start) **must be identical on every publish instance**, which is why Adobe's cluster/farm guides tell you to copy the crypto keys after cloning. Default storage is the filesystem under `crx-quickstart/launchpad/felix/bundle<N>/data` since AEM 6.3; `/etc/key` in the repository is the older location and the one the SDK note in [[aemaacs-sdk]] packages. Mismatched keys produce the "login works on one publisher, bounces on the other" ticket; mismatched `master` keys mean encrypted OSGi config values (`{...}`) fail to decrypt on the clone. Service-to-service tokens are signatures rather than MACs: [[aemaacs-server-to-server-access-tokens]].

Disambiguation rule: "MAC" next to *address, switch, ARP, Wi-Fi, OUI* is the hardware identifier; next to *key, HMAC, TLS, integrity, token* it is the authentication code.

## Cheat sheet

- MAC address = 48-bit link-layer identifier; OUI prefix names the vendor; second-lowest bit of byte 1 = software-assigned (`x2/x6/xA/xE`); `ff:…:ff` broadcast.
- Lives only on the local segment; routers replace it hop by hop; servers never see a client's MAC.
- IP → MAC via ARP (IPv4 broadcast) or ICMPv6 Neighbor Discovery (so don't block ICMPv6).
- Modern phones and laptops randomise it per Wi-Fi network; MAC filters, MAC-keyed DHCP reservations and MAC-bound licences break accordingly. Not a security control.
- Stale ARP after failover and duplicate MACs on cloned VMs are the two outages it causes; AEM 6.x licences are not MAC-bound.
- Message Authentication Code = keyed integrity tag (HMAC); AEM's encapsulated token needs the same `hmac` key on every publisher.
