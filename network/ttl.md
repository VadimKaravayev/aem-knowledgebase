# TTL: the IP hop counter, what happens when it runs out, and the other "TTLs" that are not hop counts

TTL (time to live) is a one-byte field in the IPv4 header (RFC 791) that counts **router hops, not seconds**. IPv6 renames it **hop limit** (RFC 8200) to say what it always was. Companion docs: [traceroute.md](traceroute.md) (the tool built on it), [icmp.md](icmp.md) (the message a dead packet produces), [ip-addresses.md](ip-addresses.md) (DNS, where the other TTL lives).

## The stamped-ticket picture

Every packet leaves with a ticket stamped with a number. Each router it passes through punches one hole. A router that is handed a ticket with one punch left does not forward it: it throws the packet away and mails a note back to the sender saying where it died. The only purpose is to stop a packet that has got into a **routing loop** from circulating forever; without the counter, a loop between two misconfigured routers would fill the link with the same packets until something rebooted.

## How it works on the wire

- The **sender's operating system sets the initial value**: Linux, macOS and most BSDs use 64, Windows 128, Cisco IOS and many other network devices 255. Nothing in the protocol fixes the number; it is a sysctl (`net.inet.ip.ttl` on macOS, `net.ipv4.ip_default_ttl` on Linux) or a per-socket option (`IP_TTL`).
- **Each router decrements it by exactly one** before forwarding, and recomputes the IPv4 header checksum because the header changed ([internet-checksum.md](internet-checksum.md)). Switches, Wi-Fi access points and anything else at layer 2 do not touch it; only a layer-3 hop counts. A NAT box or firewall that routes counts as a hop; a transparent bridge does not.
- **A router that would decrement to 0 drops the packet** and sends **ICMP Time Exceeded (type 11, code 0)** to the source address, carrying the first bytes of the dead packet so the sender can match it. The destination host never sees anything. Hosts receiving a packet with TTL 1 accept it; the rule is about *forwarding*.
- **Nobody tops it up.** There is no mechanism for a router to raise the value. So if the path has more hops than the starting TTL, the packet can never arrive: with TTL 50 and 100 routers in between, every packet dies at router 50, every time, and the sender sees either an ICMP Time Exceeded (if it listens and nothing filters ICMP) or just a timeout.

In practice that failure does not happen on the public internet. Paths are typically 10–25 hops and almost never above 30 (which is why traceroute stops at 30 by default), so 64 leaves a wide margin. A TTL runs out for three real reasons: a **routing loop** (the actual thing it protects against; traceroute shows the same pair of routers alternating), **software that set it low on purpose** (traceroute, MTR, some multicast scopes), or **a tunnel or overlay** that adds hidden hops without the sender knowing.

## What you can read from it

Because the starting values cluster at 64, 128 and 255, the TTL on an arriving packet tells you two things at once:

- **Roughly what sent it.** A received TTL of 52 almost certainly started at 64 (a Linux or macOS host); 116 started at 128 (Windows); 245 started at 255 (a router or a BSD-derived appliance). `ip.ttl` in Wireshark, the `ttl` field in `tcpdump -v`, the `ttl=` in `ping` output.
- **How far away it is.** 64 − 52 = 12 routers between you and it. Two replies from "the same" server with TTLs of 52 and 115 came from two different machines (a load balancer, a CDN edge versus origin, or a spoofed source). Replies whose hop count changes mid-connection mean the path changed.

Security uses the same property in reverse. **GTSM** (Generalized TTL Security Mechanism, RFC 5082): a router accepts a BGP or OSPF session only from packets that arrive with TTL 255. Since every real hop decrements it, a packet with 255 on arrival must have come from a directly connected neighbour; an attacker several hops away cannot produce one no matter what value they send. Firewalls also drop packets with TTL 0 or 1 on arrival as scan or evasion attempts (fragments crafted to expire before an IDS reassembles them).

## Seeing it

```bash
ping -c 3 example.net                      # "ttl=52": initial 64 minus 12 hops (reply direction)
traceroute example.net                     # one line per TTL value; stars = no Time Exceeded came back
sudo tcpdump -nn -v 'icmp[icmptype] == 11' # watch Time Exceeded arriving (traceroute from elsewhere, or a loop)
ping -c 1 -m 3 example.net                 # macOS: send with TTL 3 (-t on Linux); expect "Time to live exceeded" from hop 3
```

Wireshark: `ip.ttl < 5` finds probes and loop victims, `icmp.type == 11` the death notices, and the ICMP payload contains the original IP header so you can see what died and with what TTL it was sent. `ip.ttl` as a column on a server-side capture sorts clients by distance and OS family.

## The other TTLs

The name is reused for unrelated **durations in seconds**, and the collision causes real confusion:

| "TTL" | What it is | Unit |
|---|---|---|
| IP header TTL / IPv6 hop limit | Router hops remaining | hops |
| DNS record TTL | How long a resolver may cache the answer (`dig` shows it in the second column) | seconds |
| HTTP cache lifetime (`Cache-Control: max-age`, `s-maxage`, CDN "TTL") | How long a cache may serve the copy without revalidating | seconds |
| Application caches (Redis `EXPIRE`, Guava/Caffeine `expireAfterWrite`, Dispatcher `/statfileslevel` and `/enableTTL` headers) | Entry expiry | seconds |

Only the first one counts hops. When a document says "TTL" without context, the unit tells you which it means.

## What this means for AEM

The IP TTL is almost never the problem in an AEM setup, and on AEM as a Cloud Service you cannot observe it anywhere near AEM. Where the word matters is the collision: Dispatcher's `/enableTTL` honours the `Cache-Control`/`Expires` **seconds** on a cached response, the CDN's "TTL" is `s-maxage`/`Surrogate-Control` seconds, and the DNS TTL on a custom domain's CNAME decides how long a cutover to or from the Adobe CDN takes to propagate. All three are durations, and none of them has anything to do with the hop counter in this doc. The one place the hop counter helps is reading a client-side or on-prem capture: the `ip.ttl` of replies tells you whether a response came from a CDN edge a few hops away or from an origin much further off, and whether two responses that should come from the same host actually did.

## Cheat sheet

- IPv4 TTL / IPv6 hop limit = 8-bit **hop counter**, set by the sender's OS (64 Linux/macOS, 128 Windows, 255 routers), decremented by every router, never increased.
- Reaches 0 at a router → packet dropped, **ICMP Time Exceeded (11/0)** to the sender, destination never sees it. More hops than the starting TTL = the packet can never arrive.
- Internet paths are 10–25 hops; TTL expiry in the wild = routing loop, deliberate low TTL (traceroute), or hidden tunnel hops.
- Arriving TTL ⇒ sender's OS family and distance: 52 ≈ Linux 12 hops away, 116 ≈ Windows 12 hops away; different TTLs from "one" server = different machines.
- GTSM (RFC 5082): accept routing-protocol packets only at TTL 255 = sender must be adjacent.
- DNS TTL, cache `max-age`/`s-maxage`, Dispatcher `/enableTTL` are **seconds**, unrelated to the IP field.
