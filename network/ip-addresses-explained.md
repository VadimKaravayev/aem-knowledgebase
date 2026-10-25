# IP addresses, explained like you're a child

Beginner-level mental model. The AEM-relevant consequences are at the end; the dense docs are linked from there.

## The mailbox picture

Every house in the world needs a mailbox with an address, so the mail carrier knows where to bring letters. Computers work the same way: every computer, phone, or server connected to the internet gets an **IP address** — its house address on the internet. The common kind (IPv4) is four numbers separated by dots:

```
151.101.3.10
```

When your laptop asks a website "show me this page," it sends a letter that says **who it's for** (the website's IP) and **who sent it** (your IP), so the answer knows where to come back to. No return address — no reply.

## Names vs. addresses (DNS)

You type `adobe.com`, not numbers — like saying "grandma's house" instead of "17 Oak Street." A phone book called **DNS** looks up the real number-address for the name. Two phone-book entry types matter in practice:

- **A record** — "this name = this number": the name points straight at an IP.
- **CNAME record** — "this name = that *other name*": an alias; the lookup continues at the other name, which eventually ends at an A record.

A rule of the phone book: the *apex* (bare) domain like `example.com` can't be a CNAME — it must use A records — while subdomains like `www.example.com` can alias freely. That one rule is why Cloud Manager's DNS instructions differ for apex vs. subdomain ([cloud-manager-custom-domain-names.md](../cloud-service/cloud-manager-custom-domain-names.md): apex → four Fastly A records `151.101.{3,67,131,195}.10`, subdomain → `CNAME cdn.adobeaemcloud.com`).

## Private vs. public addresses

Your home Wi-Fi gives each device a *private* address that only works inside the house (like "the kids' room") — these come from reserved ranges (`192.168.x.x`, `10.x.x.x`, `172.16–31.x.x`) that the internet's mail carriers refuse to route. The whole house shares one *public* address that the outside world sees; the router rewrites letters on the way out and back (NAT), like a receptionist forwarding mail. Consequence: a server's log shows the office's one public IP for fifty employees — which is exactly what makes office-wide allow lists feasible.

## Guest lists (IP allow lists)

A building can have a doorman with a list: "only let in visitors from these addresses." That's an **IP allow list** — Cloud Manager uses them to say "only our office may reach author/preview" ([cloud-manager-environments.md](../cloud-service/cloud-manager-environments.md): Sites author+publish+preview; note the preview service ships *pre-blocked* by a default `Preview Default` list you must swap out). The classic failure: a legitimate *service* isn't on the list and gets turned away with the burglars — the Screens Services Provider starved by a publish allow list ([screens-as-a-cloud-service.md](../cloud-service/screens-as-a-cloud-service.md)) is the canonical case, fixed by letting the service in on a secret header instead of an address.

Ranges are written in **CIDR** notation: `192.168.1.0/24` means "every address whose first 24 bits match" — i.e. `192.168.1.0`–`192.168.1.255` (256 addresses). Smaller suffix = bigger block: `/32` is one address, `/16` is 65,536. Reading `/24` as "24 addresses" is the standard beginner misreading.

## Running out of numbers (IPv4 vs. IPv6)

Four dot-separated numbers give ~4.3 billion addresses — a city that ran out of street addresses years ago (NAT is the workaround that lets a whole house share one). The modern format, **IPv6**, looks like `2606:2800:220:1:248:1893:25c8:1946` and has practically infinite room. Dual reality to expect in configs: many services advertise both (DNS `A` = IPv4, `AAAA` = IPv6), and an allow list that covers only the IPv4 range silently admits/blocks differently for clients arriving over IPv6.

## One letter ≠ one conversation

Strictly, IP only delivers individual letters. "Loading a page" is a longer exchange — TCP numbers the letters and re-sends lost ones (like registered mail), and **ports** are apartment numbers inside the building: one IP, many services (`:443` HTTPS, `:4502` local AEM author, `:8023` the TarMK cold-standby sync port in [tarmk-cold-standby.md](../infrastructure-ops/tarmk-cold-standby.md)). "IP address + port" is the full "building + apartment" address a connection actually uses.
