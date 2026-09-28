# Cloud Manager SSL certificates: DV vs OV/EV, adding, lifecycle

Cloud Manager gives self-service tooling to install and manage the TLS
certificates that sit in front of an AEMaaCS environment's custom domains.
The one sentence that shapes every decision on this page: **Cloud Manager
does not issue certificates or private keys for you** unless you take the
Adobe-managed DV route; anything OV/EV comes from your own Certificate
Authority (DigiCert, GlobalSign, Entrust, etc.) and you upload it.

Companion notes: [[cloud-manager-environments]] (custom domains are a
publish/preview, Sites-program feature), [[aemaacs-architecture-overview]]
(Adobe CDN as the single front door), [[ssl-termination-sling-mapping-404]]
(what happens *behind* TLS termination on 6.5/AMS; not relevant on AEMaaCS
where the CDN and dispatcher are Adobe-run).

---

## 1. Where the certificate actually lives

```
browser ──TLS──▶ Platform TLS service ──▶ Adobe CDN (Fastly) ──▶ dispatcher/publish
                       │
                       └─ picks the cert that terminates the connection,
                          then routes by that cert + the CDN hosting the domain
```

- The **Platform TLS service** terminates TLS and routes to the customer's CDN
  service based on (a) which certificate matched the connection and (b) which
  CDN service hosts that domain.
- Several certificates can be installed per environment; the TLS layer selects
  **the most specific and most recently deployed** match for a hostname.
- **Order of operations** (self-service flow): add the custom domain in
  Domain Settings → add the SSL certificate → add the CDN configuration →
  domain goes live. A domain can be *added* before any cert exists, but it
  cannot be *associated with an environment* (CDN config) until a valid cert
  covers it.

## 2. The two management models

| | Adobe-managed **DV** | Customer-managed **OV / EV** |
|---|---|---|
| Who issues | Adobe, via Let's Encrypt | Your CA; you upload cert + key |
| Validation level | Domain Validation only | Organization / Extended Validation |
| Programs | production **and sandbox** | production programs |
| Renewal | **automatic every 3 months** until you delete it | your job; upload the renewed cert before expiry |
| Ongoing dependency | the **CNAME must stay in place** for renewal | none, beyond expiry tracking |
| Wildcards / multi-domain | per hostname set | wildcards and multi-SAN accepted; one cert can serve **multiple environments** |
| Cost/effort | zero cost, near-zero effort | CA cost, procurement lead time, renewal discipline |

### Adobe-managed DV gotcha (exam favourite)

Validation is CNAME-based. If you remove the CNAME record after the initial
issuance, the **next automatic renewal fails**, the certificate expires, and
the site goes down. Adobe's wording: "removing the CNAME record prior to
automatic certificate renewal can cause the renewal to fail ... result in
certificate expiration and service disruption." DNS clean-up tickets are the
usual culprit.

Let's Encrypt rate limits also apply: at most **5 certificates per exact
hostname set in any 7-day period**. Repeatedly deleting and re-adding the same
DV domain set during a migration rehearsal will lock you out for a week.

## 3. Customer-managed certificate requirements

**Rejected outright**
- DV certificates (use the Adobe-managed path for those).
- Self-signed certificates.
- RSA keys **larger than 2048-bit** (3072/4096 not supported at time of writing).

**Accepted**
- X.509 TLS certificate from a trusted public CA, PEM-encoded
  (`.pem`, `.crt`, `.cer`, `.cert`).
- Private key in **PKCS#8** format.
- Keys: **RSA 2048**, or EC **`prime256v1` (secp256r1)** / **`secp384r1`**.
- **ECDSA certificates are Adobe-recommended over RSA** for performance,
  security and efficiency. If the question asks "which key type should the
  architect recommend", ECDSA is the intended answer, not "RSA 4096".

**Convert what the CA gave you** (OpenSSL, all produce PEM):

```bash
# PFX / PKCS#12 bundle → PEM (cert + key, unencrypted)
openssl pkcs12 -in certificate.pfx -out certificate.cer -nodes
# P7B / PKCS#7 chain → PEM
openssl pkcs7 -print_certs -in certificate.p7b -out certificate.cer
# DER binary → PEM
openssl x509 -inform der -in certificate.cer -out certificate.pem
```

**Validate locally before uploading** so a chain error surfaces on your laptop,
not in the Cloud Manager UI:

```bash
openssl verify -untrusted intermediate.pem certificate.pem
```

## 4. Best practices and limits

- **Do not overlap coverage.** `*.example.com` plus a separate
  `dev.example.com` cert on the same environment is a support-ticket
  generator: the TLS layer chooses "most specific + most recently deployed",
  which changes silently every time either one is renewed.
- **Cap: 70 installed certificates per Cloud Manager program**, and **expired
  certificates count** toward it. Delete expired ones as part of the renewal
  runbook or you will hit the ceiling on a launch day.
- **Up to 100 SANs per certificate**, so consolidating brands/markets into
  one multi-SAN cert is the way to stay under the 70 cap.
- Prefer one OV/EV cert shared across stage + prod (allowed) over per-env
  certs when the SAN list is the same.

## 5. Adding a certificate (Cloud Manager UI)

**Role required:** Business Owner **or** Deployment Manager. Program managers
and developers cannot add, update or delete certificates.

Path: `my.cloudmanager.adobe.com` → program → side menu → **Services → SSL
Certificates** → **Add SSL Certificate**.

| Field | Adobe-managed (DV) | Customer-managed (OV/EV) |
|---|---|---|
| Certificate name | required | informational only |
| Domains | pick from **already verified** custom domains | not asked; taken from the SANs |
| Certificate | n/a | paste the leaf cert, PEM |
| Private key | n/a | paste PKCS#8 key, PEM |
| Certificate chain | n/a | paste the **intermediates (and root)** only |

- **Do not paste the leaf certificate into the chain field.** Adobe's warning:
  "do not include the new certificate in the certificate chain. Including it
  prevents the upload from completing." Same rule on update.
- **Verification timing differs.** DV: the domain must be verified *first*,
  then Adobe validates, issues and installs; the cert shows a green check
  once issued. OV/EV: domain verification runs *after* you save the cert.
- The UI validates inline and blocks Save until detected errors are fixed;
  that is where the RSA-4096, DV-cert and self-signed rejections surface.
- After Save, the next step for both types is the **CDN configuration**;
  the certificate alone does nothing.

## 6. Lifecycle: status colours, update, rename, delete

**Status list on the SSL Certificates page**

| Colour | Meaning | What to do |
|---|---|---|
| Green | valid for **≥ 14 days** | nothing |
| Orange | expires in **< 14 days** | Cloud Manager sends recurring UI notifications; renew/replace now |
| Red | **expired** | associated domains are already down; update or delete immediately |

There is no email escalation described; the signal is the UI colour plus UI
notifications, so someone must actually open Cloud Manager. Put OV/EV expiry
in an external calendar or monitoring.

**Update / replace a customer-managed cert** (three-dot menu → **View and
Update**): paste the new certificate, update the private key only if it
changed, paste the chain (again without the leaf), optionally rename, then
**Update**. It applies automatically, no pipeline run. Replacing a
still-valid cert follows the identical dialog. When several multi-SAN certs
cover the same domain and one is updated, the updated one is installed for
that domain, which is the concrete form of the "most recently deployed"
rule from section 4.

**Rename** (Adobe-managed certs, three-dot menu → **Rename**): cosmetic, for
audit clarity when many DV certs exist. Customer-managed certs are renamed
inside View and Update.

**Delete** (three-dot menu → **Delete**):
- Permanent, no undo. Save the PEM files locally first.
- **An Adobe-managed cert with active associated domains cannot be deleted**;
  the Delete entry is greyed out until every domain is removed from it.
- After confirming, **run the pipeline** to actually undeploy the
  certificate. Deleting in the UI alone leaves it installed.

**Pre-existing CDN configurations**: the page may show informational banners
about CDN configs created outside the UI (support tickets, older flow) and ask
you to migrate them into the UI for visibility. The banners take one to two
business days to disappear after migration.

## 7. Decision rule for design reviews

```
Need EV/OV green-bar assurance, or corporate PKI policy?  ──yes──▶ customer-managed OV/EV
        │ no
Sandbox program, dev/test hostnames, or "just make it HTTPS"?  ──▶ Adobe-managed DV
        (and document the CNAME as permanent infrastructure)
```

## Exam checklist

- Adobe-managed = **DV only**, Let's Encrypt, **3-month auto-renew**, needs the
  CNAME forever, works in **sandbox** programs too.
- Customer-managed = **OV/EV only**; DV and self-signed uploads are rejected.
- Key support: RSA **2048 only**; EC P-256 / P-384; **ECDSA recommended**.
- Private key must be **PKCS#8**, everything PEM.
- **70 certs max incl. expired; 100 SANs per cert**; 5 DV issuances per
  hostname set per 7 days.
- Overlapping wildcard + specific certs → non-deterministic selection ("most
  specific and most recently deployed").
- Flow: custom domain → SSL certificate → CDN configuration. DV needs the
  domain verified *before* the cert; OV/EV verifies *after* the cert is saved.
- Only **Business Owner / Deployment Manager** can add, update or delete.
- Chain field = intermediates only; **leaf in the chain field = upload fails**.
- Status: green ≥ 14 days, orange < 14 days (UI notifications), red = expired.
- Update applies automatically; **delete requires a pipeline run** to undeploy.
- DV cert with active domains **cannot be deleted** until the domains are removed.

## References
- [Introduction to SSL certificates (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/manage-ssl-certificates/introduction-to-ssl-certificates)
- [Add an SSL certificate (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/manage-ssl-certificates/add-ssl-certificate)
- [Manage SSL certificates (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/manage-ssl-certificates/managing-certificates)
- [Let's Encrypt rate limits](https://letsencrypt.org/docs/rate-limits/#new-certificates-per-exact-set-of-identifiers)
