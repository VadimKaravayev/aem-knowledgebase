# Network interface card (NIC): what the hardware does, and what it can and cannot tell you about AEM

A NIC (network interface card, network adapter, "the network card") is the piece of hardware that turns frames into signals on a cable, fibre or radio and back again. The operating system does not talk to the card directly; it talks to an **interface** (`en0` on macOS, `eth0`/`enp0s3` on Linux) that a **driver** presents on the card's behalf. Most "network" problems that are not DNS or a firewall are decided in this layer: link speed and duplex, dropped frames under load, offload features that make captures lie, and virtual NICs whose bandwidth is a quota rather than a wire. Addresses in the examples use the documentation ranges. Companion docs: [ifconfig.md](ifconfig.md) (reading the interface the card presents), [mac-addresses.md](mac-addresses.md) (the address burned into it), [mtu.md](mtu.md) (the largest frame it will carry), [tcpdump.md](tcpdump.md) and [internet-checksum.md](internet-checksum.md) (why offloading makes captures look broken).

## The loading-dock picture

If IP addresses are postal addresses and MAC addresses are the names on the doors, the NIC is the loading dock. Trucks (frames) arrive on the road (the cable). The dock checks the name on each parcel: its own name, "everyone" (broadcast), or a group it signed up for (multicast). Anything else it ignores without telling anyone. Accepted parcels go onto a conveyor belt (the receive ring buffer) and the dock rings a bell (an interrupt) so the warehouse (the kernel) comes to collect them. Outgoing parcels go the other way: the warehouse puts them on the outbound belt, the dock loads them onto trucks.

Three things follow:

- **If the belt is full, parcels fall off.** The dock cannot stop the road. A burst that arrives faster than the kernel collects is silently dropped and shows up only as an error counter.
- **The dock can do paperwork for the warehouse.** Checking and writing checksums, splitting a big shipment into truck-sized pieces, stripping VLAN labels. That is **offloading**, and it means what the warehouse hands over is not literally what goes on the road.
- **A dock has one name and one road.** More roads means more docks (several cards), or a virtual dock that stands in front of several real ones (bonding) or in front of none (loopback, VPN tunnels, container bridges).

## What the card actually does

| Layer | Job | Where it shows |
|---|---|---|
| **PHY** (physical) | Signalling on the medium; **auto-negotiation** of speed and duplex; carrier detection | `media: autoselect (1000baseT <full-duplex>)` and `status: active` in `ifconfig`; `Speed`/`Duplex`/`Link detected` in `ethtool` |
| **MAC** (link) | Framing, preamble, the **FCS** (CRC-32 trailer) check, **address filtering** (own unicast + broadcast + subscribed multicast; promiscuous mode turns the filter off) | the `ether` line; CRC errors as `Ierrs` / `rx_crc_errors`; `PROMISC` flag |
| **DMA and rings** | Descriptor rings for RX and TX; the card writes frames straight into kernel memory | `rx_missed_errors`, `rx_no_buffer_count`, `drop` in `ip -s link`; ring sizes in `ethtool -g` |
| **Interrupts** | Signal "frames waiting"; **coalescing** batches several frames per interrupt; Linux NAPI switches to polling under load | `ethtool -c`; interrupt rate in `/proc/interrupts`; one busy CPU core at 100% softirq while the rest idle |
| **Offloads** | **Checksum** (`*_CSUM`), **TSO/GSO** (card splits a 64 KB TCP chunk into MTU-sized segments), **LRO/GRO** (merges incoming segments), VLAN tag insert/strip, **RSS** (multi-queue: hash each flow to a queue and CPU) | `options=<TSO4,TSO6,PARTIAL_CSUM,…>` in macOS `ifconfig`; `ethtool -k` on Linux |
| **Driver and firmware** | The kernel module that owns the card; firmware on the card itself | `ethtool -i`; `kextstat`/`system_profiler` on macOS; driver bugs are the usual cause of "link flaps every N hours" |

The FCS deserves one sentence: it is a strong CRC over the whole frame, computed and checked **by the card**, and a frame that fails it is dropped before the kernel sees it. That is why the weak 16-bit Internet checksum in IP/TCP/UDP is tolerable ([internet-checksum.md](internet-checksum.md)): the link layer already caught almost everything.

## Interface versus card

The two are not one-to-one, and most confusion comes from assuming they are.

- **One card, several interfaces.** VLAN sub-interfaces (`en0.100`, `eth0.100`) are the same dock with several name badges; IP aliases are several addresses on one interface.
- **Several cards, one interface.** **Bonding / link aggregation** (`bond0`, LACP 802.3ad) presents one interface over two or more cards for redundancy and, for *many* flows, more bandwidth. A **bridge** (`bridge0`, `docker0`) is a software switch joining several interfaces.
- **No card at all.** `lo0` (loopback), `utun0` (VPN tunnel), `awdl0` (AirDrop), `veth` pairs for containers. They have MTUs, counters and sometimes MACs, but no PHY and no FCS, and a VPN `utun` costs 40–80 bytes of MTU on every packet ([mtu.md](mtu.md)).
- **Virtual machines.** The guest sees a card that is a program on the host: emulated hardware (`e1000`, slow, every packet trapped to the hypervisor) or **paravirtual** (`virtio-net`, VMware `vmxnet3`, AWS **ENA**), which is a direct contract with the host and several times faster per CPU cycle. **SR-IOV** hands a slice of a real card straight to the guest. Cloud instances add a twist: the "NIC" bandwidth is a **quota of the instance size**, and a figure such as "up to 10 Gbit/s" is a burst backed by credits, not a wire.
- **Naming.** macOS numbers interfaces `en0`, `en1`, … in discovery order, so a Thunderbolt dock or USB adapter can appear as `en4` on one boot and `en6` on another ([ifconfig.md](ifconfig.md) has the mapping recipe). Linux uses predictable names (`enp0s3`, `eno1`) on bare metal and often plain `eth0` inside VMs and containers.

## Seeing it

macOS has no `ethtool`; the information is spread over several tools.

```
networksetup -listallhardwareports      # Hardware Port ↔ enX ↔ hardware MAC, including docks and adapters
system_profiler SPNetworkDataType       # type (Ethernet/Wi-Fi), BSD name, media options, DHCP lease
ifconfig en0                            # options= (offloads), media (speed/duplex), status, MTU, MAC
netstat -i                              # per-interface Ipkts/Ierrs/Opkts/Oerrs/Coll counters
netstat -I en0 -w 1                     # same counters, once per second: watch errors grow under load
```

Linux puts almost everything behind `ethtool` and `ip`:

```
ethtool eth0                            # Speed, Duplex, Auto-negotiation, Link detected
ethtool -i eth0                         # driver, firmware version, bus address
ethtool -k eth0                         # offloads on/off (tx-checksumming, tcp-segmentation-offload, generic-receive-offload)
ethtool -g eth0                         # ring sizes: current vs hardware maximum
ethtool -c eth0                         # interrupt coalescing
ethtool -S eth0 | grep -iE 'drop|miss|err|no_buf'   # per-queue hardware counters: the first place to look for silent loss
ip -s link show eth0                    # RX/TX bytes, packets, errors, dropped, overruns
lspci | grep -i -E 'ether|network'      # which physical card it is
```

Reading the counters:

- **`Ierrs` / `rx_crc_errors` / `frame` growing** on a wired server port: bad cable, bad port, or a **duplex mismatch**. Not software.
- **`drop` / `rx_missed` / `rx_no_buffer` growing, errors flat**: frames arrived fine but the ring overflowed; the kernel was too slow to collect. Bigger rings (`ethtool -G`), coalescing, more queues, or less load per host.
- **`Coll` (collisions) on anything faster than 10 Mbit/s half duplex**: a duplex mismatch, because full-duplex links cannot collide.
- **`Oerrs`**: usually the link going down mid-transmit; pair it with `dmesg`/system log for link-flap messages.

## Speed, duplex and the mismatch

Auto-negotiation lets both ends agree on the fastest common speed and full duplex. The classic failure is one side **hard-coded** (an old switch port set to "100 full" by policy) and the other left on auto: the auto side cannot see the other's duplex setting, falls back to **half duplex**, and the link runs with one end expecting collisions and the other not. Symptoms: `ping` is fine, small requests are fine, anything sustained (a large package upload, a replication burst, a load test) collapses, and `Coll`/late-collision/CRC counters climb on one or both sides. Gigabit Ethernet requires auto-negotiation by standard, so this bites on legacy 100 Mbit/s ports, some management ports, and misconfigured switch templates. Fix it by setting **both** ends to auto, or both to the same fixed value; never one of each. Verify with `ethtool eth0` on Linux or the `media:` line of `ifconfig en0` on macOS, and ask the network team for the switch port's view too.

## Gotchas

- **Offloading makes captures lie, twice.** With checksum offload the kernel hands frames to the card with an unfilled checksum, so a capture on the sender shows "incorrect checksum" on every outgoing packet. With TSO the capture shows single "packets" of 20–64 KB that never existed on the wire. Both are normal on the capturing host and vanish on a capture taken from another machine ([tcpdump.md](tcpdump.md)). Turn offloads off (`ethtool -K eth0 tso off gso off gro off tx off`) only for the duration of a debugging session; they are there to save CPU.
- **Promiscuous mode does not see other hosts' traffic on a switch.** It turns off the card's address filter, but a switch still forwards each frame only to the port that owns the destination MAC. You need a mirror/SPAN port, a hub, or Wi-Fi monitor mode.
- **One flow, one queue, one CPU.** RSS spreads *flows* across queues by hashing addresses and ports, so a single TCP connection lands on one queue and one core. One huge transfer can max a core while the card is idle; many parallel connections scale. The same limit applies to bonding: a single flow never exceeds one member link.
- **Jumbo frames are all-or-nothing per segment.** A card happily configured for MTU 9000 next to a 1500 switch port produces exactly the black-hole pattern in [mtu.md](mtu.md): small things work, large transfers hang.
- **Randomised Wi-Fi MACs.** On recent macOS and mobile OSes the address `ifconfig` shows on Wi-Fi can be a per-network random one, not the card's hardware address; `networksetup -listallhardwareports` reports the hardware one. MAC-keyed allow lists and DHCP reservations break "randomly" because of this ([mac-addresses.md](mac-addresses.md)).
- **Energy Efficient Ethernet and power management** can introduce link flaps or a few milliseconds of latency after idle on some card/switch combinations; disabling EEE on the switch port is a common fix for "first request after a pause is slow".
- **The wrong virtual driver is a silent tax.** A VM on an emulated `e1000` spends several times the CPU per packet compared with `virtio-net`/`vmxnet3`, and tops out far below line rate. It works, so nobody looks.

## What this means for AEM

- **AEMaaCS**: nothing on the server side is yours. Author and publish pods run on Adobe's infrastructure, and the only NIC you own is the one in the laptop or office network reaching `*.adobeaemcloud.com`. Practical relevance: a VPN `utun` interface explains the MTU penalty on uploads, and Wi-Fi versus wired explains a flaky Universal Editor session. Cloud Manager IP allow lists see your public IP, never your card.
- **AMS / on-prem**: every internal hop rides a server NIC: Dispatcher → publish on 4503, author → publish replication, author and publish → shared datastore or NFS, TarMK cold standby sync on 8023, MongoMK replica traffic. When "replication queues block intermittently", "page loads crawl under load while CPU is idle", or "one Dispatcher is slower than its twin", read `netstat -i` or `ethtool -S` on both ends **before** touching OSGi configuration. Growing errors or drops are a network fault; a flat zero rules the card out in a minute.
- **The duplex mismatch is the AEM load-test ghost.** A publish host that passes functional tests and dies at 200 concurrent users, with the JVM and CPU healthy, is a candidate for a hard-coded switch port. Check `ethtool`/`media:` first; it is a five-second test.
- **Bonding gives redundancy, not a fatter pipe for replication.** A 5 GB package or a datastore sync is one flow, so it is bounded by one member link. Size the single link for the biggest transfer, not the aggregate.
- **Datastore and jumbo frames.** If the storage network runs MTU 9000, every AEM host's interface on that VLAN must match, and nothing with MTU 1500 may sit in the path. A binary upload that hangs at the same percentage every time is the signature.
- **Virtualised AEM.** On vSphere use `vmxnet3`, on KVM `virtio-net`, on AWS the ENA driver; an emulated adapter is a measurable throughput and CPU loss on publish tiers. On AWS also remember that a small instance's "up to" bandwidth is a burst credit; a large replication or a datastore GC can exhaust it and throttle the whole host (see [[aem-on-aws-whitepaper]] for the sizing discussion).
- **Captures while debugging Dispatcher 502s.** A tcpdump on the publish host showing bad checksums and 30 KB segments is offloading, not a corrupt AEM response. Capture on the Dispatcher side, or on a mirror port, before concluding anything about the payload.

## Cheat sheet

- NIC = PHY (signals, auto-negotiation) + MAC (framing, FCS CRC-32, address filter) + DMA rings + interrupts + offloads, owned by a driver. The OS sees an interface, not a card.
- Interface ≠ card: VLANs and aliases (one card, many interfaces), bonds and bridges (many cards, one interface), `lo0`/`utun`/`veth` (no card), VMs (a program pretending to be a card; prefer paravirtual drivers).
- Inspect: macOS `networksetup -listallhardwareports`, `system_profiler SPNetworkDataType`, `ifconfig en0`, `netstat -i`; Linux `ethtool [-i|-k|-g|-c|-S] eth0`, `ip -s link`.
- Errors growing = cable/port/duplex; drops growing with errors flat = ring overflow, kernel too slow; collisions on a full-duplex link = duplex mismatch.
- Both ends auto-negotiate, or both fixed to the same value. Never one of each.
- Offloads make local captures show bad checksums and over-MTU packets; promiscuous mode still sees only your own traffic on a switch.
- One flow = one queue = one core = one bond member. Parallelism scales, a single transfer does not.
- AEM: irrelevant on AEMaaCS beyond your own laptop and VPN; on AMS/on-prem check NIC counters and duplex before OSGi when replication, Dispatcher or datastore traffic is intermittently slow; jumbo frames must match end to end; bonding does not speed up one replication.
