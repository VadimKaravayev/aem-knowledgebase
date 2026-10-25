# ifconfig: reading an interface, and what it can and cannot tell you about AEM

`ifconfig` prints and (as root) changes the configuration of network interfaces: address, netmask, MAC, MTU, up/down state, media. On macOS and the BSDs it is still the primary tool. On Linux it is deprecated in favour of `ip` and often not installed; the Linux equivalents are given throughout. Output below was verified on macOS, Oct 2026, with addresses replaced by documentation values. Companion docs: [ip-addresses.md](ip-addresses.md), [mac-addresses.md](mac-addresses.md), [mtu.md](mtu.md), [tcpdump.md](tcpdump.md).

## The one block you must be able to read

```
en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
	options=6460<TSO4,TSO6,CHANNEL_IO,PARTIAL_CSUM,ZEROINVERT_CSUM>
	ether 00:00:5e:00:53:01
	inet6 fe80::1c2d:3e4f:5a6b:7c8d%en0 prefixlen 64 secured scopeid 0xe
	inet 192.0.2.10 netmask 0xffffff00 broadcast 192.0.2.255
	nd6 options=201<PERFORMNUD,DAD>
	media: autoselect
	status: active
```

Line by line:

- **`en0`**: interface name. macOS numbers all wired and wireless ports `enN`; which one is Wi-Fi is not visible here (see "Which en is which" below). Linux: `eth0`/`ens33`/`enp0s3`/`wlan0`.
- **`flags=8863<...>`**: the hex is the OR of the flag bits (`UP` 0x1, `BROADCAST` 0x2, `SMART` 0x20, `RUNNING` 0x40, `SIMPLEX` 0x800, `MULTICAST` 0x8000 = 0x8863). The two that matter:
  - **`UP`** = administratively enabled (someone ran `ifconfig en0 up`, or the OS did).
  - **`RUNNING`** = the link is actually working (cable in, Wi-Fi associated). **`UP` without `RUNNING` is "configured but no carrier"**: the classic unplugged cable or unassociated Wi-Fi.
- **`mtu 1500`**: the largest IP packet this link will carry. 1500 is Ethernet/Wi-Fi, 16384 on `lo0`, ~1400 on VPN `utun` interfaces. Details and the black-hole failure mode: [mtu.md](mtu.md).
- **`options=...`**: hardware offloads (`TSO4/6` TCP segmentation offload, `*_CSUM` checksum offload). Operationally relevant in one case: **captures taken on this host show wrong or absent TCP checksums** because the NIC fills them in after tcpdump sees the frame. Not a network fault.
- **`ether`**: the MAC address. Linux prints `link/ether`. Randomised on modern Wi-Fi clients ([mac-addresses.md](mac-addresses.md)).
- **`inet6 ... %en0 prefixlen 64 secured scopeid 0xe`**: an IPv6 address. `fe80::` prefix = link-local, exists on every interface even when IPv6 is "not used". `%en0` is the zone (which interface), needed when you address a link-local target. `secured` = RFC 7217 stable privacy address (not derived from the MAC). A second `inet6` line starting with a global prefix and `temporary` is the rotating outbound privacy address.
- **`inet 192.0.2.10 netmask 0xffffff00 broadcast 192.0.2.255`**: the IPv4 address. **macOS prints the netmask in hex**: `0xffffff00` = `255.255.255.0` = `/24`. Count the `f`s: each is 4 one-bits. `ifconfig -f inet:cidr en0` prints `/24` instead (verified; the `-f` display-format flag is macOS-only). Linux `ip addr` prints CIDR directly.
- **`nd6 options`**: IPv6 Neighbor Discovery settings (`DAD` = duplicate address detection on). Ignore unless debugging IPv6.
- **`media: autoselect`** / **`status: active`**: the link layer. On wired ports `media` shows the negotiated speed and duplex (`1000baseT <full-duplex>`); a `100baseTX <half-duplex>` on a server is an autonegotiation failure and a real throughput problem. `status: inactive` = no carrier.

## The forms you actually type

```bash
ifconfig                       # every interface, including down and virtual ones
ifconfig en0                   # one interface
ifconfig -l                    # just the names, one line (macOS)
ifconfig -lu                   # names of interfaces that are UP (macOS)
ifconfig -a                    # all, same as bare ifconfig on macOS; on Linux bare ifconfig hides down interfaces
ifconfig -v en0                # macOS extras: link quality, uplink rate, interface type "Wi-Fi", router mode
ifconfig -f inet:cidr,inet6:cidr en0   # CIDR netmasks instead of hex (macOS)
```

Changes need root and are **not persistent**: a reboot, a DHCP renewal or a Wi-Fi reconnect undoes them. Use them for diagnosis, then fix the real config (System Settings / `networksetup` on macOS, netplan/NetworkManager on Linux).

```bash
sudo ifconfig en0 down && sudo ifconfig en0 up     # bounce the interface (drops every connection on it)
sudo ifconfig en0 mtu 1400                         # test an MTU hypothesis without touching the VPN config
sudo ifconfig en0 alias 192.0.2.99 netmask 255.255.255.0   # add a second IP (talk to a device on a foreign subnet)
sudo ifconfig en0 -alias 192.0.2.99                # remove it
sudo ifconfig en0 ether 00:00:5e:00:53:99          # spoof the MAC (why MAC filtering is not security)
sudo ifconfig en0 promisc                          # receive frames for other MACs; tcpdump does this for you
```

Linux `ip` equivalents: `ip addr` / `ip -br addr` (brief table), `ip link set eth0 up|down`, `ip link set eth0 mtu 1400`, `ip addr add 192.0.2.99/24 dev eth0`, `ip -s link show eth0` for counters.

## Which `en` is which (macOS)

`ifconfig` does not say. On a laptop with Thunderbolt docks the list is long (`en0`–`en6`, `bridge0`, `ap1`, `awdl0`, `llw0`, `utun0`–`utun4` all appeared on the verified machine). To map names to hardware:

```bash
networksetup -listallhardwareports    # "Hardware Port: Wi-Fi / Device: en0 / Ethernet Address: ..."
route -n get default                  # which interface carries the default route right now
ifconfig -v en0 | grep type           # "type: Wi-Fi"
```

Virtual interfaces you will see and can ignore: `lo0` loopback; `gif0`/`stf0` tunnel stubs; `awdl0`/`llw0` AirDrop/Apple peer-to-peer; `ap1` the personal-hotspot access point; `bridge0` for Thunderbolt bridging or VMs; `utunN` VPNs and iCloud Private Relay; `anpiN` Apple debug ports. **If `route -n get default` names a `utun`, all traffic is going through a VPN**, which changes MTU, source IP and often DNS.

## Companion commands for the questions ifconfig raises

| Question | Command |
|---|---|
| What is my IP, nothing else | `ipconfig getifaddr en0` (macOS), `hostname -I` (Linux) |
| Where did the IP come from (DHCP lease, server, router, DNS handed out) | `ipconfig getpacket en0` (macOS): shows `yiaddr`, `server_identifier`, `lease_time`, `router`, `domain_name_server` |
| Which gateway and interface for a destination | `route -n get 203.0.113.20` (macOS), `ip route get 203.0.113.20` (Linux) |
| Full routing table | `netstat -rn -f inet` (macOS), `ip route` (Linux) |
| Packet and error counters per interface | `netstat -i`, `netstat -ib -I en0` (bytes), `ip -s link` (Linux) |
| Which DNS resolvers are in use | `scutil --dns` (macOS), `resolvectl status` (Linux) |
| Which MAC answers for a LAN IP | `arp -a` ([arp.md](arp.md)) |

`netstat -i` **`Ierrs`/`Oerrs` or `Coll` that keep growing** on a server interface mean a bad cable, duplex mismatch or driver problem. A number that is non-zero but static since boot is noise.

## Gotchas

- **Two meanings of "down".** `ifconfig` not listing an address means the interface has none (DHCP failed), which is different from the interface lacking `RUNNING` (no link), which is different from lacking `UP` (disabled). Read the flags before the address.
- **Hex netmask misreads.** `0xfffffe00` is `/23`, not `/24`. When a host "can reach half the subnet", compare masks across hosts in CIDR form.
- **169.254.x.x means DHCP failed.** The host self-assigned a link-local IPv4 address (APIPA). It will talk to the LAN but has no gateway.
- **Multiple `inet` lines** = aliases. A server with a VRRP or keepalived virtual IP shows it as an alias on the active node only. After a failover, `ifconfig` on both nodes tells you which one really holds the VIP; stale ARP elsewhere is the next suspect ([arp.md](arp.md)).
- **Linux `ifconfig` is the old net-tools package.** It truncates long interface names, hides secondary addresses that `ip addr` shows, and may be absent. Prefer `ip`, and translate when reading old runbooks.
- **Changes do not survive**, and bouncing an interface on a remote host over the same interface locks you out. Use a console or an out-of-band path.

## What this means for AEM

- **AEMaaCS**: nothing. You never see an interface of an AEM pod. `ifconfig` is only useful on *your* side: confirm which interface and VPN your browser or pipeline runner is using before you blame a Cloud Manager IP allow list (the allow list sees the public IP of whatever path `route -n get default` names).
- **AMS / on-prem**: the first command on an AEM, Dispatcher or load-balancer host when "it cannot reach X".
  - Is the interface `UP` and `RUNNING`, with the IP you expect and the mask the other hosts use? A publisher on `/23` next to a Dispatcher on `/24` reaches only half the segment.
  - Is the **MTU** what the path needs? Replication or asset uploads that hang between zones while small requests work are MTU/PMTUD problems ([mtu.md](mtu.md)); test with `ifconfig <if> mtu 1400` before changing anything permanent.
  - Does `netstat -i` show growing errors on the publish or Dispatcher NIC? Then the "intermittent 502s" are layer 1, not Apache.
  - Which node holds the **virtual IP** after a Dispatcher or LB failover? Alias present = active node.
  - `inet` bound to `0.0.0.0` in AEM's own config is a different question: AEM listens on all interfaces by default; restrict with the Jetty `org.apache.felix.http` host property, not with `ifconfig`.
- **Local development**: a Docker Dispatcher or an SDK author on a laptop that moves between Wi-Fi, a dock and a VPN changes IP and interface; `ifconfig` plus `route -n get default` explains why `host.docker.internal` or a hard-coded IP in a renders config stopped working.

## Cheat sheet

- `flags`: need both `UP` (enabled) and `RUNNING` (link). `status: active` confirms carrier.
- `netmask 0xffffff00` = `/24`; use `-f inet:cidr` to stop converting.
- `ether` is the MAC; `inet6 fe80::` exists everywhere and is not "IPv6 in use".
- `mtu` per link; `utun` present in `route -n get default` = VPN path.
- Changes need root and vanish on reboot; bouncing an interface drops every connection on it.
- Linux: use `ip addr`, `ip link`, `ip route`, `ip -s link`.
