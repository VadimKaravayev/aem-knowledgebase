# TLS: how the handshake, keys and certificates actually work, and what you can see of it in Wireshark

TLS (Transport Layer Security) is the layer between TCP and the application protocol that makes the connection private and authenticated. HTTPS is HTTP spoken through TLS; the same layer protects SMTP, IMAP, LDAP, PostgreSQL and anything else that wants it. The current version is **TLS 1.3 (RFC 8446)**; TLS 1.2 (RFC 5246) is still everywhere; TLS 1.0 and 1.1 are formally deprecated (RFC 8996) and disabled in current browsers and JDKs. "SSL" is the pre-1999 name and survives only in library and tool names (`openssl`, `mod_ssl`, `SSLKEYLOGFILE`). Companion docs: [tcp.md](tcp.md) (what TLS sits on), [quic.md](quic.md) (TLS 1.3 embedded in a transport), [ip-addresses.md](ip-addresses.md) (DNS, where the hostname being authenticated comes from).

## The sealed-envelope-with-a-passport picture

TCP is a phone call anyone on the line can listen to. TLS is two strangers on that call who first agree on a shared secret by shouting public numbers at each other in a way that lets each compute the same key but lets no eavesdropper do so (Diffie-Hellman), then one of them shows a passport issued by an authority the other already trusts and signs the whole conversation so far with it (the certificate and its signature), and from then on everything is spoken in a code only the two of them hold. Three properties fall out:

- **Confidentiality**: after the handshake an observer sees only record sizes and timing.
- **Authentication**: the server proved it holds the private key matching a certificate that a trusted CA issued for the name the client asked for. Clients are authenticated the same way only when the server asks (mutual TLS).
- **Integrity**: every record carries an authentication tag; a flipped bit is detected and the connection is torn down with an alert, never silently accepted.

What TLS does *not* hide: the server's IP address and port, the hostname (sent in the clear in the Server Name Indication extension unless Encrypted ClientHello is in use), how much data flowed and when, and, in TLS 1.2, the server's certificate.

## Where it sits: the record layer

TLS is a stream of **records** inside the TCP byte stream. Each record has a 5-byte header: content type (1 byte), legacy version (2 bytes, always `0x0303` = "TLS 1.2" on the wire even for 1.3), length (2 bytes, up to 2¹⁴ bytes of plaintext). Content types: 20 change_cipher_spec (vestigial in 1.3, sent only to fool old middleboxes), 21 alert, 22 handshake, 23 application_data. In TLS 1.3 every record after the ServerHello is type 23 on the outside and carries its real type as the last plaintext byte inside the encryption, so a middlebox cannot even tell a handshake message from data. Records are independent of application message boundaries: one HTTP response may span many records, and one record may hold the tail of one response and the head of the next.

The cipher does **AEAD** (authenticated encryption with associated data): AES-GCM or ChaCha20-Poly1305, producing ciphertext plus a 16-byte tag, with a per-record nonce derived from a sequence number so replayed or reordered records fail authentication. TLS 1.3 removed every non-AEAD option (CBC, RC4, 3DES) and every non-ephemeral key exchange (static RSA), which is why its cipher suite names are just `TLS_AES_128_GCM_SHA256`, `TLS_AES_256_GCM_SHA384`, `TLS_CHACHA20_POLY1305_SHA256`: the suite names only the record cipher and the hash for the key schedule; key exchange group and signature algorithm are negotiated by separate extensions. A TLS 1.2 suite name encodes all of it: `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256` = ephemeral elliptic-curve DH key exchange, RSA-signed, AES-128-GCM records, SHA-256 PRF.

## The TLS 1.3 handshake, step by step

```
client                                                     server
ClientHello  ───────────────────────────────────────────►
  random, cipher_suites, extensions:
  supported_versions [1.3,1.2], supported_groups [x25519, P-256],
  key_share {x25519: g^a}, signature_algorithms, server_name (SNI),
  alpn [h2, http/1.1], psk_key_exchange_modes, [pre_shared_key]
             ◄───────────────────────────────────────────  ServerHello
                                      random, chosen suite, supported_versions=1.3,
                                      key_share {x25519: g^b}
             ──── both derive handshake keys from g^ab; everything below is encrypted ────
             ◄───────────────────────────────────────────  {EncryptedExtensions}  (ALPN choice, etc.)
             ◄───────────────────────────────────────────  {Certificate}          (leaf + intermediates)
             ◄───────────────────────────────────────────  {CertificateVerify}    (signature over transcript hash)
             ◄───────────────────────────────────────────  {Finished}             (HMAC over transcript)
{Finished}   ───────────────────────────────────────────►
[application data] ◄────────────────────────────────────► [application data]
             ◄───────────────────────────────────────────  {NewSessionTicket}     (for resumption later)
```

One round trip from ClientHello to the client's first application byte (TCP's handshake adds one more; QUIC folds them together).

1. **ClientHello.** 32 random bytes, the suites and versions it accepts, and crucially a **key_share**: the client guesses which Diffie-Hellman group the server will pick (almost always x25519) and sends its public value *up front*. If the server supports none of the offered groups it answers **HelloRetryRequest** naming one, costing an extra round trip; Wireshark shows this as a second ClientHello. The `legacy_version` field says 1.2 and the real list is in `supported_versions`, because enough middleboxes broke on unknown version bytes that 1.3 had to disguise itself as 1.2.
2. **ServerHello.** Its own random, the one suite it chose, its key share for the same group. Both sides now compute the shared secret `g^ab` and run it through the **key schedule**: HKDF-Extract and HKDF-Expand steps (RFC 5869) that mix in a running **transcript hash** of every handshake message so far. Out come separate *handshake traffic secrets* for each direction, from which the record keys and IVs are derived. From this point the server encrypts.
3. **EncryptedExtensions** carries everything negotiated that is not needed to establish keys: the ALPN choice (`h2` or `http/1.1`, which is how HTTP/2 is selected), server-name acknowledgement, early-data acceptance.
4. **Certificate**: the server's leaf certificate followed by the intermediates needed to reach a root the client has; the root itself is normally omitted. **CertificateVerify**: a signature, with the leaf's private key, over the transcript hash. This is the actual proof of identity: anyone can replay a certificate, only the key holder can sign *this* conversation's transcript. **Finished**: an HMAC over the transcript with a key derived from the handshake secret, proving the server derived the same keys and saw the same messages (which is what defeats downgrade attacks: tampering with the ClientHello's suite list changes the transcript).
5. **Client Finished** closes the loop; if the server asked for a client certificate (`CertificateRequest` after EncryptedExtensions), the client's `Certificate` and `CertificateVerify` go first.
6. Both sides derive the **application traffic secrets** from the master secret and switch keys. Either side can later send `KeyUpdate` to ratchet forward.
7. **NewSessionTicket**, sent by the server any time after its Finished, hands the client an opaque ticket plus a resumption PSK. Next time the client puts `pre_shared_key` in its ClientHello (binder = HMAC proving it knows the PSK), the server skips certificate and signature, and the client may send **0-RTT early data** encrypted under a PSK-derived key *with* the ClientHello. 0-RTT is replayable (an attacker can re-send the captured ClientHello and early data), so servers must only accept idempotent requests in it or answer `425 Too Early`.

**Forward secrecy** is built in: the long-term key only *signs*; the record keys come from ephemeral DH values that are discarded. Capturing traffic today and stealing the server's private key next year decrypts nothing. This is also why Wireshark's "give me the server's RSA key" decryption only ever worked for TLS 1.2 static-RSA suites.

**Downgrade protection**: a 1.3-capable server that ends up negotiating 1.2 writes a fixed sentinel (`DOWNGRD\x01`) into the last 8 bytes of its ServerHello random; a 1.3-capable client that sees it aborts.

## What TLS 1.2 does differently

Two round trips, and the server's certificate goes in the clear:

```
ClientHello ─►   ◄─ ServerHello, Certificate, ServerKeyExchange (ECDHE params, signed), ServerHelloDone
ClientKeyExchange (client's ECDHE value), ChangeCipherSpec, Finished ─►
◄─ ChangeCipherSpec, Finished     then application data
```

The client cannot send its key share in the ClientHello because it does not yet know which group the server will use, hence the extra round trip. With the (now disabled) RSA key exchange the client instead encrypted a random premaster secret with the server's public key, so no forward secrecy. Resumption used a session ID the server remembered or a session ticket (RFC 5077) encrypted under a server key; both skipped the certificate exchange on a second handshake but gave no 0-RTT. Renegotiation existed (and was a vulnerability magnet); 1.3 removed it in favour of KeyUpdate and post-handshake authentication.

## Certificates: what is actually checked

A certificate is an **X.509 v3** structure (DER-encoded ASN.1; PEM is base64 of the same with `-----BEGIN CERTIFICATE-----` lines) that binds a public key to a set of names, signed by an issuer. Fields that matter:

- **Subject Alternative Name (SAN)**: the DNS names (and optionally IPs) the certificate is valid for; wildcards cover one label (`*.example.com` matches `www.example.com`, not `example.com` or `a.b.example.com`). The old Common Name is ignored by browsers and by Java's hostname verifier since around 2017.
- **Issuer**, **validity** (not-before / not-after; public CAs are limited to 398 days today, being shortened in steps towards 47 days by 2029 under a CA/Browser Forum ballot), **key usage** and **extended key usage** (`serverAuth`, `clientAuth`), **basic constraints** (`CA:TRUE` for intermediates and roots), **Authority / Subject Key Identifier** (how chains are linked), **Authority Information Access** (URL of the issuer's certificate and of its OCSP responder), **CRL distribution points**, **Signed Certificate Timestamps** (Certificate Transparency proof; browsers require them for public certs).

**Chain building.** The client takes the leaf, finds a certificate whose subject matches the leaf's issuer (from the ones the server sent, or from its own store, or, in browsers only, by fetching the AIA URL), verifies the signature, and repeats until it reaches a certificate that is in its **trust store** (the OS store, Mozilla's bundle, the JDK's `cacerts`). Then it checks every link's dates, that intermediates are CAs, that the leaf's SAN matches the requested hostname, that EKU allows server use, and, optionally, revocation via **OCSP** (or an OCSP response the server **staples** into the handshake via the `status_request` extension, which avoids the client contacting the CA) or a CRL. Any failure is a fatal alert such as `unknown_ca` (48), `certificate_expired` (45), `bad_certificate` (42). The three failures that account for most tickets: **missing intermediate** (browsers hide it via AIA fetching, `curl`, Java and most servers do not), **name mismatch** (connecting by IP or by an alias not in the SAN), and **private CA not in the client's trust store** (corporate TLS-inspection proxies re-sign every site with an internal root, so everything works in the managed browser and nothing works in a container or JVM that lacks that root).

**Mutual TLS** reverses the check: the server sends `CertificateRequest`, the client presents a certificate chained to a CA the server configured, and signs the transcript. It authenticates the *client machine or service*, not a user, and is the usual way service-to-service calls are locked down without bearer tokens.

## Extensions you will meet

- **SNI (server_name)**: the hostname, in the clear, so one IP can serve many certificates. Omitting it (old clients, connecting by IP, some Java code paths) gets the server's default certificate and then a name mismatch. **Encrypted ClientHello (ECH)** encrypts the real SNI under a public key published in the domain's `HTTPS` DNS record and shows an outer, shared "front" name; deployed by some CDNs and browsers, not universal.
- **ALPN**: the client lists application protocols (`h2`, `http/1.1`), the server picks one in EncryptedExtensions. HTTP/2 is only ever negotiated this way over TLS. `h3` is *not* chosen here; it is discovered via `Alt-Svc` ([quic.md](quic.md)).
- **supported_versions, supported_groups, signature_algorithms, key_share**: the negotiation described above. A server that supports only P-256 facing a client that sent only an x25519 share triggers HelloRetryRequest.
- **status_request** (OCSP stapling), **signed_certificate_timestamp**, **pre_shared_key / early_data** (resumption, 0-RTT), **renegotiation_info** (1.2 only), **extended_master_secret** (1.2 hardening).

## Alerts and how connections end

An alert record is two bytes: level (1 warning, 2 fatal) and description. `close_notify` (0) is the polite end and should precede the TCP FIN; many implementations skip it and just close. Fatal ones you will see in logs: `handshake_failure` (40, no common suite/group/version), `bad_certificate` (42), `certificate_expired` (45), `unknown_ca` (48), `decrypt_error` (51, wrong Finished or signature, often a middlebox that modified the handshake), `protocol_version` (70, client or server too old), `unrecognized_name` (112, SNI the server has no certificate for). In TLS 1.3 every alert after ServerHello is encrypted, so Wireshark shows it as an Application Data record of exactly 19 bytes of payload (2 alert bytes + 1 inner type + 16 tag), typically right before a FIN or RST: that fixed size is your only hint that the server rejected something rather than finishing normally.

## Seeing it in Wireshark

Wireshark dissects TLS on port 443 automatically and on other ports via *Decode As → TLS*. Field names are the display-filter names (Wireshark 4.x; the protocol was renamed from `ssl` to `tls` in 3.0).

**What is visible without keys.** The ClientHello, the ServerHello, the alert and record framing, and in TLS 1.2 the certificate chain. That is enough for most diagnosis:

```
tls.handshake.type == 1                       # ClientHello: who connected, offering what
tls.handshake.type == 2                       # ServerHello: what was chosen
tls.handshake.extensions_server_name          # SNI: which hostname each connection asked for
tls.handshake.extensions.supported_version    # real version (ServerHello's version field lies: 0x0303)
tls.handshake.ciphersuite                     # suites offered / chosen
tls.handshake.extensions_alpn_str             # h2 vs http/1.1 offered and chosen
tls.handshake.type == 11                      # Certificate (readable only in 1.2)
x509ce.dNSName                                # SANs inside a 1.2 certificate
tls.alert_message.desc                        # unencrypted alerts (1.2, or before ServerHello)
tls.record.content_type == 23 && tls.record.length == 19   # 1.3 encrypted alert candidates
tls.handshake.type == 2 && !tls.handshake.extensions.supported_version   # ServerHello that is really 1.2
```

**Reading a handshake.** Click the ClientHello and compare the *Supported Versions*, *Cipher Suites*, *Supported Groups* and *Key Share* lists with the ServerHello's choices. A second ClientHello in the same `tcp.stream` after a short ServerHello means HelloRetryRequest (Wireshark labels it "Hello Retry Request"). A ServerHello followed immediately by an alert or by FIN with no Certificate is a server rejecting the client (version or suite mismatch, unrecognised SNI). A Certificate message followed by a *client* alert is the client rejecting the server (trust, name, expiry). A complete handshake followed by application data and then an RST from the server is not TLS at all; look at the application. *Statistics → Conversations* plus `tls.handshake.extensions_server_name` as a custom column gives "which hostnames did this machine talk to", which is what you actually get from a capture of HTTPS traffic without keys.

**Timing.** `tcp.time_delta` ([tcp.md](tcp.md)) between the ClientHello and the client's first Application Data record is the TLS cost of the connection; compare with `tcp.analysis.initial_rtt` to see whether it is one round trip (1.3, or 1.2 resumed), two (1.2 full, or 1.3 with HelloRetryRequest) or dominated by server-side work (certificate lookup, OCSP fetch).

**Decrypting.** Only with the session keys, because of forward secrecy. Browsers, `curl` and anything built on OpenSSL, BoringSSL or NSS write them to the file named by the `SSLKEYLOGFILE` environment variable (`CLIENT_RANDOM <hex> <secret>` lines in 1.2, `CLIENT_HANDSHAKE_TRAFFIC_SECRET` etc. in 1.3); point *Preferences → Protocols → TLS → (Pre)-Master-Secret log filename* at it and the stream decodes to HTTP/1.1 or HTTP/2. `openssl s_client -connect host:443 -servername host -keylogfile keys.log` does the same for a hand-made connection. Java has no built-in key log; a key-logging Java agent is the usual route. Server-side, nginx and Apache do not key-log; capture on the client, or terminate TLS somewhere you can log. Wireshark's older "RSA keys list" (server private key) decrypts only TLS 1.2 RSA key-exchange sessions, which no sane server offers any more.

**Command-line companions.**

```bash
openssl s_client -connect example.net:443 -servername example.net -showcerts </dev/null
                                                    # chain as sent, negotiated version/suite, alerts; add -tls1_2 to force
openssl s_client ... | openssl x509 -noout -text    # decode the leaf: SAN, dates, issuer, EKU
openssl s_client ... -status                        # is OCSP stapled?
curl -vI https://example.net                        # "SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384", ALPN result, cert subject/issuer/dates
curl --cacert corp-root.pem ...                     # test with a private root without touching the system store
nmap --script ssl-enum-ciphers -p 443 example.net   # what the server accepts, graded
keytool -printcert -sslserver example.net:443       # what the JDK sees; keytool -list -cacerts for what it trusts
```

## Gotchas

- **The ServerHello says version 1.2.** Always. The negotiated version is in the `supported_versions` extension; a tool or filter that reads `tls.record.version` or `tls.handshake.version` will report every 1.3 connection as 1.2.
- **"Works in the browser, fails in curl / Java / the server-side call"** is a missing intermediate nine times in ten. Browsers fetch it from the AIA URL or have it cached; nothing else does. Fix the server's chain, do not add the intermediate to the client's trust store.
- **Resumed handshakes have no Certificate message.** If you need to see the certificate in a capture, make the client forget its tickets (private window, fresh `curl`) or filter for `tls.handshake.type == 11` across the whole file and find the first full handshake to that host.
- **Clock skew is a certificate error.** A container or VM with a wrong clock gets `certificate_expired` or `not yet valid` for a perfectly good certificate. Check `date` before the CA.
- **Connecting by IP address sends no SNI** (SNI must be a hostname) and almost always yields the wrong certificate. Use the hostname, or `--resolve host:443:ip` in curl to test a specific backend with the right SNI.
- **Corporate TLS inspection** rewrites every chain to an internal root. Symptoms: works in the managed browser, `unknown_ca` everywhere else, and `openssl s_client` shows an issuer you do not recognise. The capture shows the proxy's ServerHello, not the real server's.
- **A capture of TLS without keys is metadata only**, but SNI, certificate (1.2), record sizes and timing leak a lot; do not treat "it's encrypted" as "there is nothing to see".
- **JDK defaults move.** Current JDKs (11.0.x, 17, 21) disable TLS 1.0 and 1.1 and anon/NULL/RC4/3DES suites via `jdk.tls.disabledAlgorithms` in `java.security`; a legacy endpoint that only speaks 1.0 fails with `No appropriate protocol (protocol is disabled or cipher suites are inappropriate)`, and the right fix is on the endpoint.
- **0-RTT and retries.** If a client library retries a POST that was sent as early data and rejected with 425, it must resend over the 1-RTT connection, not treat 425 as a server error.

## What this means for AEM

- **TLS terminates in front of AEM, never in it.** On AEM as a Cloud Service the Adobe-managed CDN holds the certificates (managed or customer-uploaded through Cloud Manager, see `cloud-manager-ssl-certificates`), negotiates versions, suites and HTTP/2 with the visitor, and talks to the Dispatcher over a connection you do not configure; there is no Cloud Manager or Dispatcher setting for protocol versions or ciphers. On AMS or on-prem the same job is done by a load balancer or by Apache `mod_ssl` on the Dispatcher host. Either way AEM itself sees plain HTTP and learns the original scheme only from `X-Forwarded-Proto`, which is the root of the Sling-mapping 404 and wrong-scheme-in-links problems documented in `ssl-termination-sling-mapping-404`.
- **AEM as a TLS client is a JVM.** Outbound calls (`HttpClientBuilderFactory`, replication to a publish behind HTTPS, IMS, translation and commerce connectors) validate the remote chain against the JDK's `cacerts`, so `javax.net.ssl.SSLHandshakeException: PKIX path building failed` means the remote sent an incomplete chain or uses a private CA. On AEMaaCS you cannot edit `cacerts`; the remote must present a publicly trusted chain, or the integration must be one that reads AEM's Granite Global Trust Store (SAML signing certificates and code written to use `KeyStoreService` do; generic `HttpClient` code does not unless you wire it). On 6.5 you can also import the private root into `cacerts`, with the usual operational cost of re-doing it after every JDK update.
- **Version and SNI mismatches surface as application errors.** A connector failing with `No appropriate protocol` is the JDK refusing TLS 1.0/1.1 on a legacy endpoint; a `hostname verification failed` on a publish or external host usually means the client connected by IP or an alias not in the SAN. Reproduce from the AEM host with `openssl s_client -servername` or `keytool -printcert -sslserver` before touching AEM configuration.

## Cheat sheet

- TLS = records (5-byte header: type, legacy version `0x0303`, length) inside TCP; AEAD-encrypted after the handshake; 1.3 = RFC 8446, 1.2 = RFC 5246, 1.0/1.1 dead.
- **1.3 handshake, 1 RTT**: ClientHello (random, suites, `supported_versions`, `key_share` guess, SNI, ALPN) → ServerHello (suite, `key_share`) → keys from ECDH + HKDF over the transcript hash → {EncryptedExtensions, Certificate, CertificateVerify = signature over transcript, Finished} → {Finished} → data. HelloRetryRequest = wrong group guess, +1 RTT. **1.2 = 2 RTT, certificate in clear, ServerKeyExchange carries the signed ECDHE params.**
- **Forward secrecy**: the certificate key only signs; record keys come from ephemeral DH, so a stolen private key decrypts nothing recorded. Only session keys (`SSLKEYLOGFILE`) decrypt a capture.
- **Resumption** = PSK from NewSessionTicket; **0-RTT** is replayable → idempotent only / `425 Too Early`.
- **Certificate check**: build chain leaf → intermediates (server-sent) → root (client store); verify signatures, dates, `CA:TRUE` on intermediates, SAN matches hostname (CN ignored, wildcard = one label), EKU, revocation (OCSP, ideally stapled). Top failures: missing intermediate (browsers hide it via AIA), name mismatch (IP / alias), private root not in the store (corporate inspection, JVM `cacerts`).
- **Suites**: 1.3 names only cipher + hash (`TLS_AES_128_GCM_SHA256`); group and signature are separate extensions. 1.2 names everything (`TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`).
- **SNI** in clear (ECH encrypts it; needs an `HTTPS` DNS record); **ALPN** picks h2 vs http/1.1; h3 is via Alt-Svc, not ALPN.
- **Alerts**: `close_notify` 0, `handshake_failure` 40, `bad_certificate` 42, `certificate_expired` 45, `unknown_ca` 48, `decrypt_error` 51, `protocol_version` 70, `unrecognized_name` 112; in 1.3 they are encrypted = a 19-byte Application Data record before the FIN.
- **Wireshark**: `tls.handshake.type == 1/2/11`, `tls.handshake.extensions_server_name`, `tls.handshake.extensions.supported_version` (the ServerHello version field always says 1.2), `tls.handshake.extensions_alpn_str`, `tls.alert_message.desc`; decrypt via *Preferences → TLS → (Pre)-Master-Secret log*; resumed handshakes carry no certificate. CLI: `openssl s_client -servername -showcerts`, `curl -vI`, `keytool -printcert -sslserver`.
- **AEM**: TLS ends at the CDN/LB, AEM sees HTTP + `X-Forwarded-Proto`; as a client AEM is a JVM trusting `cacerts` (`PKIX path building failed` = chain/private CA; AEMaaCS cannot edit `cacerts`, use a public chain or the Granite truststore where supported); `No appropriate protocol` = legacy TLS 1.0/1.1 endpoint.
