# Security — OAuth/OIDC, RBAC, Managed Identity, secrets

General system-design reference (not AEM-specific). Adobe IMS is a full worked example of these patterns — tie-ins at the end.

## OAuth 2.0 / OIDC (what each actually is)

- **OAuth 2.0 = delegated authorization** ("let app X act on resource Y with scope Z"), **OIDC = authentication layered on top** (adds the **ID token** — a JWT about *who logged in* — and `/userinfo`). "Login with OAuth" without OIDC is the historical bug: an access token proves authorization to call an API, not the user's identity.
- Token roles: **access token** (short-lived, sent to the resource server — opaque or JWT), **refresh token** (long-lived, sent only to the token endpoint, must be stored like a credential; rotate on use — reuse detection reveals theft), **ID token** (consumed by the client only — never send it to APIs as auth).
- Grants that still matter: **Authorization Code + PKCE** (all user-facing flows — SPA, native, web; PKCE replaces the implicit flow, which is dead: tokens in URL fragments leak via history/referrer), **Client Credentials** (service-to-service, 2-legged — no user), **Device Code** (TVs/CLIs). Resource Owner Password grant is deprecated — it teaches users to type passwords into third parties.
- Validation on the resource server, where the real vulns live: check **signature** (fetch keys via JWKS, handle rotation), **`iss`**, **`aud`** (a valid token for a *different* API must fail — the classic cross-service replay), **`exp`**, and reject `alg: none`/algorithm-confusion (pin expected algorithms). JWT revocation doesn't exist — keep access tokens short (minutes) and revoke at the refresh boundary, or use opaque tokens + introspection where instant revocation is a requirement.
- Cache tokens until near-expiry; fetching a fresh token per call is how you meet the IdP's rate limiter ([adobe-ims-authentication-types](../cloud-service/adobe-ims-authentication-types.md): ~24h S2S tokens, "cache them or get throttled").

## RBAC (and its edges)

- Model: user → **role** → permissions; authorize against roles/permissions, never against user IDs in code. Key design choices: roles are **per-tenant/scope** in multi-tenant systems ("admin *of what*"), permissions are fine-grained verbs (`invoice:read`), roles are the bundling layer — check **permissions** in code so roles can be reshaped without code changes.
- **Role explosion** is the scaling failure: "admin-but-not-billing-in-region-X" breeds hundreds of roles. Past that point you need **ABAC/policy-based** authorization (attributes + rules — OPA/Cedar-style engines) or **ReBAC** (relationships: "editor of *this* document" — Google Zanzibar / SpiceDB model; the right shape for document/content systems).
- Enforcement rules: authorize **on the server, on every request** (UI hiding is UX, not security); default-deny; re-check on resource access, not just at login (role changes must take effect before token expiry — short tokens or a revocation check); **never trust client-asserted roles** — a `role` field in an unverified header or token-without-signature-check is a privilege-escalation CVE template.
- The most common real-world authz bug is not RBAC design but **IDOR/BOLA** — object-level checks missing ("can this user see order 42", not just "can this user call GET /orders"). Object-level authorization is OWASP API #1 for a reason.

## Managed Identity / workload identity

- The pattern (Azure **Managed Identity**, AWS **IAM roles**, GCP **service accounts + workload identity**): the platform *issues* the workload's credentials at runtime via instance metadata — code asks the local metadata endpoint for a token; **no secret exists to store, leak, or rotate**. This eliminates the bootstrap problem ("where does the first credential live?") for everything running on that cloud.
- Hierarchy of auth for service-to-service, best first: platform workload identity → OIDC federation (e.g. GitHub Actions presents its OIDC token to AWS/Azure — CI without stored cloud keys) → short-lived certs/mTLS (SPIFFE) → OAuth client credentials with a vaulted secret → long-lived API keys (last resort, with rotation) → passwords in env vars (incident pending).
- System vs user-assigned (Azure): system-assigned dies with the resource (1:1, simplest), user-assigned is shareable across resources and survives them (fleet of identical workers).
- Traps: metadata endpoints are an **SSRF target** (an SSRF hole in your app can mint your identity's tokens — this is how Capital One happened; enforce IMDSv2/network rules); over-granting the workload identity recreates the leaked-key blast radius without the key; local dev can't use it, so code needs a credential-chain abstraction (`DefaultAzureCredential`-style) rather than if/else auth paths.

## Secrets

- Ladder: hardcoded (never — git history is forever; a committed secret is **rotated, not deleted**) → env vars (better; still leak via crash dumps, `/proc`, debug endpoints, and child processes) → **secret manager** (Vault/AWS SM/Azure KV: central audit, access control, rotation, versioning) → **no secret at all** (managed identity, the actual goal).
- Rotation is the design constraint people skip: dual-secret (old+new valid during rollout) is what makes rotation zero-downtime — single-valid-secret systems force a synchronized deploy, so nobody rotates, so the leak from 2019 still works. (IMS does multi-secret rotation for this reason.)
- Runtime hygiene: never log secrets (structured logging + redaction filters; the "log the whole config object at startup" incident is universal), don't put them in URLs (logged everywhere), scope them (per-service, per-env — one secret for everything = one leak for everything), and detect leaks (pre-commit scanners like gitleaks, plus canary tokens).
- Secret-manager gotcha: fetching at startup and caching forever means rotation doesn't propagate — re-fetch on auth failure or TTL.

## AEM tie-ins

- IMS OAuth Server-to-Server **is** client-credentials with product-profile-based authorization and multi-secret rotation; the JWT/service-account EOL is a live migration case — [adobe-ims-authentication-types](../cloud-service/adobe-ims-authentication-types.md). Cloud Manager's "one product profile = the credential's role" is RBAC for machine identities ([cloud-manager-api-cli-sdks](../cloud-service/cloud-manager-api-cli-sdks.md)).
- AEMaaCS secret handling: `$[secret:NAME]` env vars (deploy-restart caveats, not versioned — [aemaacs-osgi-configuration](../cloud-service/aemaacs-osgi-configuration.md)), CryptoSupport key packaging ([aemaacs-sdk](../cloud-service/aemaacs-sdk.md)), secret-in-header edge allow-listing ([screens-as-a-cloud-service](../cloud-service/screens-as-a-cloud-service.md)), and "secret at the edge, not in clientlibs" as a stated Edge Functions use case ([edge-functions](../cloud-service/edge-functions.md)).
- Oak's path-based ACLs + `rep:glob` are closer to ReBAC/attribute territory than flat RBAC — see [repoinit-acls-on-apps-and-libs](../cloud-service/repoinit-acls-on-apps-and-libs.md).
