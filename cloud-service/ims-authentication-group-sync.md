# IMS authentication and group sync into AEMaaCS author

Sources: [IMS Support for AEMaaCS](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/security/ims-support) + the [IMS user groups video tutorial](https://experienceleague.adobe.com/en/docs/experience-manager-learn/cloud-service/accessing/adobe-ims-user-groups) ("logical sets of users exposed to AEM, to define fine-tuned permissions"). The Admin Console layer above this is [admin-console-product-profiles.md](admin-console-product-profiles.md); the credential/token layer is [adobe-ims-authentication-types.md](adobe-ims-authentication-types.md).

## Scope — the boundary that decides architectures

IMS login covers **author-side humans only**: "Author, Admin, and Dev users". It does **not** serve "external end users of customer sites like site visitors", and **Publish has no IMS login at all** — site-visitor authentication on publish is your own problem (SAML handler / CUG / custom — [sso-authentication-handlers.md](../infrastructure-ops/sso-authentication-handlers.md)). Any design that routes site visitors through Adobe IMS is wrong by scope.

Flow: OAuth between AEM and IMS — "the user is redirected to IMS and optionally to the customer IDP for SSO and then redirected back to AEM" (Federated ID case). The AEM-side IMS configuration is **auto-provisioned with the environment** — there is nothing to set up in AEM itself. A **local admin login survives** as the break-glass path independent of IMS availability.

## Getting people into Admin Console (three roads)

1. **Manual UI** creation — small teams.
2. **CSV bulk upload**.
3. **User Sync Tool (UST)** — fetches users from the org's Active Directory, diffs against Admin Console, "calls the Adobe User Management API so the Admin Console is synchronized… **the change flow is entirely one way**" (AD → Adobe; nothing writes back).

## Group sync and the permission pattern

- Sync is **login-triggered**: when a user authenticates, "their group memberships are synced" into AEM — i.e. membership changes land **on that user's next login**, not instantly; there is no background full sync to wait for.
- The sanctioned permission model is **inheritance via local groups**: add the synced IMS group *as a member of* an AEM local group that owns the ACLs (doc example: `AEM-GRP_008` added to `DAM Users` "inherits the permissions and privileges of DAM Users"). Two tiers: **IMS group = who** (managed in Admin Console/AD), **AEM local group = what** (ACLs, managed in AEM, shippable as repoinit — [repoinit-acls-on-apps-and-libs.md](repoinit-acls-on-apps-and-libs.md)).
- Corollary: don't hang ACLs on individual synced users or directly on product-profile groups — profiles gate *login* ([admin-console-product-profiles.md](admin-console-product-profiles.md)), IMS user groups carry *who*, local groups carry *rights*. Three layers, each owned by a different team.

## Profile mapping and its sharp edge

"The **AEM Administrators_xxx** profile grants Administrator privileges in the associated AEM instance; **AEM Users_xxx** adds regular users." The warning here is even stronger than the onboarding journey's: **renaming the AEM Administrators product profile removes administrator rights from every user assigned to it** — the name itself is load-bearing in the mapping.

## Doc gaps worth knowing exist

The page is silent on: exact session/logout behavior, login-token refresh, whether a *removal* from a group needs re-login (assume yes, same trigger), and service-account handling (service users are repo-local, never IMS — [aem-connector-implementation-guidelines.md](../java-osgi-build/aem-connector-implementation-guidelines.md) / repoinit). Treat those as "verify on an instance" rather than settled.

Exam tells: "configure IMS login for site visitors on publish" = wrong by scope; "permissions stop working after team moved groups in AD" = login-triggered sync + one-way UST flow; "admin rights vanished for everyone" = someone renamed the AEM Administrators profile.

Docs-based; tutorial page is video-backed (code 39150), its substance folded in above.
