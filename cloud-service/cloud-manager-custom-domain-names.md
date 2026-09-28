# Cloud Manager custom domain names: Domain Settings, verification, DNS

Custom domains on AEMaaCS are added in Cloud Manager under **Services →
Domain Settings**. The page is short, but the ordering rules around it are
where people get burned: the domain, the certificate, the domain mapping and
the DNS cutover are four separate steps, and the **DNS record is the last
one** because on the Adobe CDN it is both the ownership proof and the traffic
switch.

Companion notes: [[cloud-manager-ssl-certificates]] (DV vs OV/EV, key rules,
lifecycle), [[cloud-manager-environments]] (custom domains = Sites programs,
publish and preview only, never author), [[cloud-manager-program-types]]
(sandbox programs have no custom domains).

---

## 1. Requirements

- Role: **Business Owner or Deployment Manager**.
- A CDN in front: Fastly (Adobe-managed) or your own provider. **Even with
  the Adobe-managed CDN you must still register the domain in Cloud
  Manager**; the CDN alone knows nothing about it.
- Adobe's page lists "add the SSL certificate first" as a requirement while
  the Add-SSL page lists "add the domain first". Both are true for different
  paths; see the order-of-operations table below.
- **www and apex are two domains.** `example.com` and `www.example.com` must
  be added as separate entries.
- Enter the bare hostname: no `http://`, `https://` or spaces.

## 2. Order of operations, by certificate type

```
                     Adobe-managed DV                    Customer-managed OV/EV
                     ────────────────                    ──────────────────────
1. Domain Settings   Add Domain → Create                 Add Domain → Create
2. Verify dialog     pick "Adobe managed (DV)"           pick "Customer managed (OV/EV)"
3. Proof             CNAME/A records → click Verify      click OK (nothing to prove yet)
                     (hours: DNS propagation)
4. Certificate       add DV cert, choose the             add OV/EV cert (cert + key + chain);
                     now-verified domain                 domain flips to Verified afterwards
5. Domain mapping    map domain → environment/tier       same
6. DNS cutover       already done in step 3              CNAME/A records now
```

- **DV**: the domain must be **Verified before** the DV cert can be
  requested (the Add-SSL dropdown only offers verified domains). Verification
  is the CNAME/A record itself, which also routes traffic. So Adobe says this
  step is "always done after testing is complete and you are ready to go
  live". Under an Adobe-managed CDN, DV certs are only permitted for sites
  using **ACME validation**.
- **OV/EV**: the Verify dialog is a formality (OK). The domain is marked
  Verified only **after the certificate is added**, because the cert's SANs
  are the ownership evidence.
- **Bring-your-own everything**: if you use your own OV/EV (or DV) cert on
  your **own CDN**, skip Cloud Manager's SSL page entirely and go straight
  to **Add a Domain Mapping**.
- Status changes appear in the Domain Settings table; see Adobe's "Check
  custom domain name status" page.

## 3. Which certificate wins for a domain

Same rule as on the SSL page, restated for domains: the domain is served by
the **most specific valid certificate**; if several certs carry the same
domain, the **most recently updated** one is chosen. Adobe's recommendation is
simply to have **no overlapping domains** across certificates.

## 4. DNS records (Adobe CDN / Fastly)

| Domain shape | Record | Value |
|---|---|---|
| Subdomain (`www.customdomain.com`) | **CNAME** | `cdn.adobeaemcloud.com` |
| Apex (`example.com`) | **A** ×4 on `@` (or ALIAS/ANAME if the provider has it) | `151.101.3.10`, `151.101.67.10`, `151.101.131.10`, `151.101.195.10` |

- A CNAME cannot live at the zone apex, hence the four A records. Providers
  with ALIAS/ANAME records can point the apex at the CNAME target instead.
- Prerequisites Adobe lists before touching DNS: know your registrar/DNS
  host, have (or borrow) edit rights, and have the domain already verified in
  Cloud Manager.
- If you run a **non-Adobe CDN**, these values do not apply; configure the
  domain in that CDN and Cloud Manager only needs the domain mapping.

### "Register before you advertise"

Adobe's warning, worth quoting in reviews: configure DNS **only after the
domain mapping has been added successfully**, so Cloud Manager already knows
the domain before any request for it arrives. Two reasons:

1. **Outage avoidance.** A CNAME or A record routes *all* traffic for the
   name immediately. If the target isn't provisioned or tested, users see
   errors or nothing.
2. **Domain takeover.** An unclaimed hostname pointing at a shared CDN
   endpoint can be claimed by another tenant on the same platform. Registering
   in Cloud Manager first closes that window.

For the DV path this collides with "verification needs the CNAME". Practical
reading: add the domain and (if possible) the mapping in Cloud Manager first,
then flip the CNAME at go-live, then Verify and request the DV cert. Never
leave a CNAME to `cdn.adobeaemcloud.com` dangling for a domain that is not
registered in Cloud Manager.

## 5. The `_aemverification` TXT record (authorization, not routing)

Alongside the CNAME/A records there is a **third DNS record** in the custom
domain story, and the current Adobe UI page no longer shows it: the source
markdown of "Add a custom domain name" still contains the whole section, but
it has been **commented out since the June 2026 revision**, and the old
"Add a TXT record" URL now redirects to that page. It remains live in the
**Cloud Manager API**, in the Adobe tutorial (Sept 2026), in community error
threads ("Domain verification is failed, can not find txt record"), and in
older exam material, so know it.

**What it is for.** Adobe's wording: a TXT record "authorizes a domain to be
hosted in a CDN service ... authorizes Cloud Manager to deploy the CDN service
with the custom domain and associate it with the backend service. This
association is entirely under your control ... may be granted and withdrawn.
The TXT record is specific to the domain and the Cloud Manager environment."
So unlike the CNAME, the TXT record **routes nothing**; it is a revocable
proof-of-control token binding *this hostname* to *this program + environment*.

**Format**

| Domain | Record name | TXT value |
|---|---|---|
| `example.com` | `_aemverification.example.com` | `adobe-aem-verification=example.com/<programId>/<envId>/<uuid>` |
| `www.example.com` | `_aemverification.www.example.com` | `adobe-aem-verification=www.example.com/<programId>/<envId>/<uuid>` |

Real-world shape (Adobe tutorial, WKND):

```
_aemverification.wknd.enablementadobe.com. 3600 IN TXT
  "adobe-aem-verification=wknd.enablementadobe.com/105881/991000/bef0e843-9280-4385-9984-357ed9a4217b"
```

- Copy the value **verbatim from the Verify domain dialog**; it embeds the
  program ID, environment ID and a random UUID, so it cannot be predicted or
  reused across environments. Moving a domain from stage to prod means a new
  TXT value.
- It was the verification step of the **customer-managed (OV/EV)** path in
  the previous UI; the DV path proved ownership via CNAME/A record (see
  section 2). In the current UI the OV/EV path is verified by the uploaded
  certificate instead.
- Check propagation before clicking Verify:

```bash
dig _aemverification.example.com -t txt
```

  Expected output is the exact value from Cloud Manager. Google DNS-over-HTTPS
  is a handy second opinion when your resolver caches an old answer.
- Adobe's own note (commented out too): the TXT entry and the CNAME/A record
  **can be set at the same time** on the authoritative DNS server to save a
  propagation round-trip; that does not override "register before you
  advertise" for the traffic-carrying record.

**Same thing via the Cloud Manager API** (still documented):

```
POST /api/program/{programId}/domainNames/validate
     { "name": "www.example.com", "environmentId": 2, "certificateId": 1 }
  → { "dnsTxtRecord": "adobe-aem-verification=www.example.com/53/2/<uuid>",
      "dnsZone": "example.com." }

POST /api/program/{programId}/domainNames
     { name, environmentId, certificateId, dnsTxtRecord, dnsZone }
  → domain resource with status "not_verified" and HAL links
      .../domain-name/{id}/verify   .../domain-name/{id}/deploy
```

API limitation stated by Adobe: custom domain names apply only to
**production programs with Sites enabled** (matches the "no custom domains
in sandbox" rule).

## 6. Design-review checklist

- Custom domains: **Sites programs**, **publish and preview tiers**, not
  author, not sandbox.
- Two entries for apex + www; decide the redirect direction separately.
- The Adobe CDN CNAME target is a single shared hostname; per-environment
  routing is done by the domain mapping and the certificate, not by DNS.
- DV chosen? Then the CNAME is permanent infrastructure (renewal every 3
  months validates through it; see [[cloud-manager-ssl-certificates]]).
- Own CDN? Cloud Manager still gets the domain + mapping, but not the cert.
- Go-live runbook: domain → cert → mapping → DNS, and the DNS change is the
  cutover, so schedule it as one.

## Exam checklist

- Domain Settings is under **Services** in Cloud Manager; Business Owner or
  Deployment Manager only.
- Adobe-managed CDN users **still add the domain** in Cloud Manager.
- Verify dialog asks for intended cert type; **DV → prove via CNAME/A record,
  wait hours; OV/EV → OK, verified once the cert lands**.
- Own cert + own CDN → skip SSL page, go to Domain Mapping.
- Subdomain → **CNAME to `cdn.adobeaemcloud.com`**; apex → **four A records
  (151.101.x.10)**.
- **www and non-www are separate domains.**
- "Register before you advertise": DNS last, after the mapping, to avoid
  outages and **domain takeover**.
- Overlapping certs: most specific wins, then most recently updated.
- **TXT record** `_aemverification.<domain>` =
  `adobe-aem-verification=<domain>/<programId>/<envId>/<uuid>`: revocable
  authorization tying the hostname to one environment; **routes nothing**.
  Check with `dig <name> -t txt`. Hidden from the current UI page, still in
  the API (`dnsTxtRecord` + `dnsZone`) and older exam questions.

## References
- [Add a custom domain name, GitHub source with the commented-out TXT section](https://github.com/AdobeDocs/experience-manager-cloud-service.en/blob/main/help/implementing/cloud-manager/custom-domain-names/add-custom-domain-name.md)
- [Cloud Manager API: Adding custom domain names (developer.adobe.com)](https://developer.adobe.com/experience-cloud/cloud-manager/guides/api-usage/adding-custom-domain-names)
- [Tutorial: Custom domain name with Adobe CDN (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-learn/cloud-service/content-delivery/custom-domain-name-with-adobe-managed-cdn)
- [Add a custom domain name (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/custom-domain-names/add-custom-domain-name)
- [Check custom domain name status (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/custom-domain-names/check-domain-name-status)
- [Add a domain mapping (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/domain-mappings/add-domain-mapping)
- [Add an SSL certificate (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/manage-ssl-certificates/add-ssl-certificate)
