# The Internet checksum: what it is, how IP/ICMP/TCP/UDP compute it, and why it is not security

A checksum is a small number computed from a block of data and sent along with it, so the receiver can recompute it and detect that something changed in transit. IP, ICMP, TCP and UDP all use the same 16-bit recipe (RFC 1071), so one explanation covers all four. Companion docs: [icmp-explained.md](../network/icmp-explained.md) and [ip-addresses-explained.md](ip-addresses-explained.md).

## The idea

You read a long number aloud over a bad phone line, then say "and the digits add up to 47". The listener adds up what they wrote down. If they get 47 the copy is probably right; if not, something was misheard and they ask again. The sum is the checksum. It does not fix anything, it only lets the receiver know to discard.

## The recipe (RFC 1071)

1. Treat the data as a sequence of **16-bit words**. If the length is odd, pad with one zero byte for the calculation only.
2. **Add them in ones-complement arithmetic**: whenever the sum overflows past 16 bits, take the carry and add it back at the low end ("end-around carry").
3. **Invert every bit** of the result. That value goes in the checksum field. While computing, the checksum field itself is treated as zero.

Verification is the same sum including the checksum field. Because the sender stored the inverted sum, a correct packet always totals **`0xFFFF`** (all ones). Anything else means a bit changed somewhere.

Worked example with two words (verified in Python):

```
  0x4500
+ 0x0054
--------
  0x4554          sum
  0xBAAB          inverted: this is the checksum

Receiver: 0x4500 + 0x0054 + 0xBAAB = 0xFFFF  ✓
```

```python
def internet_checksum(words):
    s = sum(words)
    while s >> 16:                     # end-around carry
        s = (s & 0xFFFF) + (s >> 16)
    return (~s) & 0xFFFF
```

## Why it was designed this way

- **Cheap.** Pure addition, fast in 1981 software, and still trivial for hardware. It is also **incrementally updatable** (RFC 1624): when a router decrements TTL by one, it adds one to the IP checksum rather than re-summing the header. NAT does the same when it rewrites an address or port.
- **Order-independent.** Ones-complement addition commutes, so word order does not matter. Convenient for hardware, but also a weakness: **two swapped 16-bit words are not detected**.
- **Weak by modern standards.** Detects every single-bit error and most random corruption, but errors that cancel (one word +1, another −1) pass. Ethernet's CRC-32 at the link layer is far stronger and catches almost everything in practice; the Internet checksum is an end-to-end safety net for corruption that happens *inside* routers and hosts, where the CRC has already been stripped.
- **Not security.** Anyone who can alter the bytes can recompute the checksum. Integrity against an attacker comes from TLS (a MAC) or IPsec, never from this field.

## What each protocol covers

| Protocol | Checksum covers | Notes |
|---|---|---|
| **IPv4 header** | The 20+ byte IP header only, **not the payload** | Recomputed (incrementally) at every hop because TTL changes |
| **ICMPv4** | The whole ICMP message, header + data | **No pseudo-header**, so the outer IP addresses are not protected |
| **TCP** | Header + data + a **pseudo-header** (src IP, dst IP, protocol, TCP length) | Pseudo-header catches a segment delivered to the wrong host or protocol. Mandatory |
| **UDP (IPv4)** | Header + data + pseudo-header | **Optional**: a sender may put `0` to mean "not computed". A computed value of 0 is sent as `0xFFFF` so it is not mistaken for "none" |
| **IPv6** | **No header checksum at all** | IPv6 relies on the link-layer CRC plus the transport checksums |
| **ICMPv6, TCP, UDP over IPv6** | Message + an **IPv6 pseudo-header** | All **mandatory**, including UDP |

The pseudo-header is not transmitted. Both ends build it from the IP header they already have and feed it into the sum.

## Consequences you will actually hit

- **NAT and ICMP errors.** Because ICMPv4's checksum excludes the outer IP addresses, a NAT can rewrite them freely. But an ICMP *error* carries a copy of the offending packet's IP header inside its data, and that quoted header is covered. A NAT that rewrites the inner copy must patch the ICMP checksum too. Early boxes got this wrong, and hosts silently discarded every ICMP error as corrupt, which looked like a PMTUD black hole. See [icmp-explained.md](../network/icmp-explained.md).
- **"Checksum incorrect" in packet captures on the sender.** Modern NICs do **checksum offload**: the kernel leaves the field blank (or partial) and the card fills it in after the point where `tcpdump`/Wireshark captures. Outgoing packets therefore look broken on the sending machine and fine everywhere else. It is a capture artifact, not an error. Wireshark: *Preferences → Protocols → TCP → Validate checksum* is off by default for this reason.
- **Silent drops, not errors.** A receiver that computes a mismatch discards the packet and tells nobody. TCP retransmits after a timeout; UDP and ICMP simply lose the datagram. A link that corrupts occasionally shows up as unexplained retransmissions and latency, never as an error message.
- **UDP checksum `0` on IPv4.** Allowed, and a few legacy appliances still send it. On IPv6 it is forbidden, and a middlebox that strips checksums from tunneled traffic (some VXLAN setups) is a classic "works on v4, breaks on v6" cause.
- **Reusing the recipe.** Because it is additive and commutative, the Internet checksum is a bad choice for anything you design yourself where reordering matters. Use CRC-32 for corruption detection, a keyed HMAC for integrity against tampering.

## What this means for AEM

Nothing in AEM computes or configures any of this; it lives in kernels and NICs. It matters when you read packet captures while debugging Dispatcher, replication or CDN problems:

- Outgoing "bad checksum" flags on the capturing host are offload noise. Ignore them.
- Stalled replication or asset uploads over a tunnel with occasional retransmissions and no errors anywhere is consistent with a corrupting link (checksum discards) as well as with PMTUD; capture on *both* ends to tell them apart.
- Integrity of content in flight between Cloud Manager, the CDN and clients comes from TLS, not from these checksums.

## Cheat sheet

- Checksum = detect corruption, not fix it, not prevent tampering.
- Internet checksum: 16-bit words, ones-complement sum with end-around carry, bitwise invert; receiver sums everything and expects `0xFFFF`.
- Cheap, incrementally updatable (TTL decrement, NAT), order-blind, catches all single-bit errors, misses cancelling pairs and swapped words.
- IPv4 header: header only, re-done per hop. ICMPv4: whole message, no pseudo-header. TCP/UDP: message + pseudo-header; UDP optional on v4. IPv6: no header checksum, transport checksums mandatory.
- "Incorrect checksum" on your own outgoing packets in a capture = NIC offload, not a bug.
