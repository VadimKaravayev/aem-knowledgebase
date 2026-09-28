# Adobe IMS authentication types for integrations (Developer Console)

Every integration that calls an Adobe API, including the Cloud Manager API
and the Adobe services AEM itself talks to (Analytics, Target, Launch,
I/O Events, Assets APIs), authenticates against **Adobe Identity Management
System (IMS)** with a credential created in a **Developer Console project**.
The old "Adobe I/O Authentication Overview" (API Key / OAuth / Service
Account JWT) is superseded; this note follows the current guide and records
the one fact that breaks legacy setups: **Service Account (JWT) credentials
are end-of-life**.

Companion notes: [[cloud-manager-api-cli-sdks]] (a consumer of these
credentials), [[app-builder]], [[exposing-apis-to-external-systems]] (the
reverse direction: others calling AEM).

---

## 1. The four authentication types

| Type | Credential(s) | Whose data | Consent | Use in AEM land |
|---|---|---|---|---|
| **Server to server** (2-legged) | **OAuth Server-to-Server** | your application's and **your organisation's** data | none; governed by **product profiles** assigned to the credential | Cloud Manager API/CLI, AEM → Analytics/Target/Launch, App Builder backends, I/O Events |
| **User** (3-legged) | OAuth **Web App**, **Single Page App**, **Native App** | an Adobe **end-user's** data | user signs in and consents to scopes | apps acting as a Creative Cloud or Experience Cloud user |
| **API key** | **API Key** (no secret, no tokens) | anonymous/public services | none; protected by **allowed origins** | Express Embed SDK, PDF Embed, Adobe Stock, API Mesh |
| **Admin** | **Enterprise Web App** (Technology Partner Program only) | an **enterprise customer's** data | the customer **admin** consents once; a technical account is created in the customer org | partner-built click-to-install apps |

Rule of thumb for design questions: *no human in the loop* → server to
server; *acting as a person* → user auth; *only need to identify the app* →
API key; *you are a partner shipping to many customer orgs* → admin.

## 2. OAuth Server-to-Server, the one you will actually use

- **Grant**: OAuth 2.0 `client_credentials`. One POST, no certificates:

```bash
curl -X POST 'https://ims-na1.adobelogin.com/ims/token/v3' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'client_id={CLIENT_ID}&client_secret={CLIENT_SECRET}&grant_type=client_credentials&scope={SCOPES}'
```

- **Token lifetime**: usually **24 hours** (`expires_in`, seconds). **Cache
  and reuse** until expiry; Adobe **throttles** integrations that mint tokens
  on every call.
- **Authorisation** is not in the token request: it comes from the **product
  profiles** attached to the credential in the Developer Console (and
  manageable by admins in **Admin Console → Users → API credentials**). No
  profile, no data. Name the credential; the name is what the admin sees.
- **Secret hygiene**: the credential supports **multiple client secrets**, so
  rotation is zero-downtime: add a new secret → deploy it → check the
  *last used* timestamp of the old one → delete the old one (irreversible).
  Rotation is also available via the **I/O Management API**
  (`/console/organizations/{orgId}/credentials/{credentialId}/secrets`,
  needs the `read_client_secret` and `manage_client_secrets` scopes).
- Token generation must run on a **secure backend**; never in a browser or
  mobile app. Use a standard OAuth library (Spring Security, Passport,
  Authlib).

## 3. Service Account (JWT) is dead: timeline and migration

| Date | What happened |
|---|---|
| 1 May 2023 | deprecation announced |
| 3 Jun 2024 | no new JWT credentials can be created |
| **30 Jun 2025** | **end of life**: certificates can no longer be refreshed |
| **1 Mar 2026** or certificate expiry | Adobe **auto-converts** remaining JWT credentials to OAuth S2S, breaking anything still doing the JWT exchange |

Why it was replaced: JWT needed a public certificate + private key pair
rotated **every year**, a three-step token dance across UI and terminal, and
a private key on disk. OAuth S2S needs none of that.

Migration is two steps and zero downtime: in the project, open the JWT
credential and click **add equivalent OAuth Server-to-Server credential**
(same client ID, technical account, APIs, scopes and product profiles; tokens
are identical), switch the application to the `client_credentials` call, then
delete the JWT credential. Find stragglers with the Developer Console project
filter *Attention Required → Has Service Account (JWT) credential*.
Auto-generated projects were migrated by Adobe.

**AEM impact**: any **Adobe IMS Configuration** in AEM (Tools → Security →
Adobe IMS Configurations) for Analytics, Target, Launch or Asset Compute that
was created with a certificate/private key is a JWT integration and must be
recreated on OAuth S2S. Same for CI jobs calling the Cloud Manager API with
a `KEY`/private-key parameter: switch to client ID + secret.

## 4. User authentication details worth knowing

- 3-legged flow: sign-in button → Adobe authorize endpoint with scopes →
  consent screen → redirect back with `code` → exchange at the IMS token
  endpoint → access token (+ **refresh token** only with the
  `offline_access` scope; store it, rotate on every refresh).
- Credentials start **In Development**: only listed **beta users** can sign
  in and the redirect URI is freely editable. **In Production**, anyone can
  sign in and redirect changes are restricted; some APIs require Adobe
  approval to promote.
- Redirect URIs: a default URI plus an optional **pattern**; Adobe only
  redirects to what matches.

## 5. API key details worth knowing

- Contains **no secret** and **cannot mint tokens**; it identifies the app,
  nothing more.
- Protection is the **`Origin` header** check against up to **five**
  allow-listed domains (wildcards for subdomains, non-privileged ports OK).
  Server-side callers have no Origin, which is why this type only fits
  browser-embedded experiences and anonymous APIs.

## 6. Admin authentication details worth knowing

- Solves "customer had to paste their own S2S secret into a partner app".
  The partner has **one** credential; each customer admin clicks *Connect
  with Adobe*, consents, and a **technical account is created in the customer
  org**. The partner mints tokens with its own client ID/secret plus the
  customer `org_id`.
- The partner backend must validate `id_token`, `state` and `nonce` on the
  redirect. Admins assign product profiles and can **revoke** on the Adobe
  Exchange manage page; existing tokens die within an hour.

## Exam checklist

- Machine-to-machine integration with Adobe APIs → **OAuth Server-to-Server**,
  `client_credentials`, product profiles govern access.
- **JWT / Service Account credentials are EOL (30 Jun 2025)**; certificate
  rotation questions are legacy, answer is "migrate to OAuth S2S".
- Tokens last ~**24 h**; cache them; Adobe throttles token spam.
- Secrets rotate with overlap (add, switch, verify last-used, delete).
- Acting as a user → 3-legged OAuth, refresh token needs `offline_access`.
- API key = identification only, guarded by **allowed origins**.
- Partner app for many customer orgs → **Enterprise Web App** / admin
  consent, TPP members only.

## References
- [Authentication guide (Adobe Developer Console)](https://developer.adobe.com/developer-console/docs/guides/authentication/)
- [Server to server authentication](https://developer.adobe.com/developer-console/docs/guides/authentication/ServerToServerAuthentication/)
- [OAuth Server-to-Server implementation guide (source)](https://github.com/AdobeDocs/adobe-dev-console/blob/main/src/pages/guides/authentication/ServerToServerAuthentication/implementation.md)
- [Migrating from Service Account (JWT) to OAuth Server-to-Server (source)](https://github.com/AdobeDocs/adobe-dev-console/blob/main/src/pages/guides/authentication/ServerToServerAuthentication/migration.md)
- [Legacy Adobe I/O Authentication Overview (superseded)](https://developer.adobe.com/developer-console/docs/guides/#!AdobeDocs/adobeio-auth/master/AuthenticationOverview/AuthenticationGuide.md)
