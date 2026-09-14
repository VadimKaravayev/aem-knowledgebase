# AEM SSO / External Identity Integration (SAML & Generic SSO Handlers)

How AEM lets authors (or publish-side users) log in via an external identity management system (IdM/IdP) instead of local AEM credentials — no custom authentication code required in the standard case.

---

## The problem it solves

A customer already has a corporate identity system (Okta, Azure AD/ADFS, PingFederate, SiteMinder, Oracle Access Manager, etc.) and wants "log in once, get into AEM automatically" for authors — instead of separate AEM local accounts.

AEM doesn't build a custom login flow for this per project. It exposes a pluggable extension point — the **Sling `AuthenticationHandler`** — that runs before the normal login form and asks "do I already know who this user is?" Two OOTB implementations cover almost every real case, and both are **configuration, not code**.

## The two OOTB handlers

| Handler | When to use | How it works |
|---|---|---|
| **SAML Authentication Handler** | Customer's IdP speaks SAML 2.0 (ADFS, Okta, PingFederate, most enterprise IdPs) | AEM acts as a SAML **Service Provider**. Unauthenticated request → redirect to IdP login → IdP returns a signed SAML **assertion** → AEM validates it (trusted cert) → creates the AEM session. Configure IdP metadata URL/cert, entity ID, attribute mapping (e.g. which SAML attribute maps to the AEM user ID/group). |
| **Generic SSO Authentication Handler** | Authentication already happened **before** the request reaches AEM — a reverse proxy or web agent (SiteMinder, Oracle Access Manager agent, a custom auth proxy) sits in front and stamps an HTTP header (e.g. `SM_USER: jane.doe`) on every request | AEM just reads the trusted header and creates the session. No token validation inside AEM — trust boundary is "only the proxy can set this header," so the proxy must strip/overwrite any client-supplied copy of that header at the edge. |

Both are configured via OSGi config (e.g. `com.adobe.granite.auth.saml.SamlAuthenticationHandler` / the generic SSO handler factory config) and path-scoped like any other `AuthenticationHandler` (`path` property, e.g. `/content/...` or `/`).

## Why the tempting wrong answers are wrong (exam framing)

- **"Implement the generic SSO Authentication Handler interface"** — implies writing your own handler from scratch. It already exists; you *configure* it, you don't reimplement it.
- **Sling Authentication *Requirements* (OSGi HTTP Service)** — this only decides *which paths require login at all*. It has zero involvement in *how* identity is established.
- **Custom JAAS Login Module** — a much lower-level Java security mechanism. Technically possible, but not the supported/intended path for enterprise SSO when SAML/generic-SSO handlers already exist. Reach for it only when no OOTB handler fits (e.g. a bespoke non-HTTP-header, non-SAML protocol).

**One-line mental model:** *something outside AEM vouches for the user (a signed SAML assertion, or a proxy-set header) → the AuthenticationHandler's only job is to trust and read that vouching, then map it to an AEM user/group.*

## User provisioning still needs an answer

Authenticating *who* the user is (SSO) is separate from *what groups/permissions* they get in AEM. Typical pairing: SAML attribute (or header value) → group mapping at first login (JIT provisioning), or sync via a **user synchronization** mechanism (e.g. an LDAP/AD sync alongside SSO) so group membership stays current without manual AEM group edits.
