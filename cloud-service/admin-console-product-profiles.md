# Admin Console, identity types, and AEMaaCS product profiles

From the AEMaaCS onboarding journey ([Admin Console](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/onboarding/journey/admin-console), [Cloud Manager profiles](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/onboarding/journey/assign-profiles-cloud-manager), [AEM profiles](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/onboarding/journey/assign-profiles-aem)). The [Admin Console](https://adminconsole.adobe.com) is "the central place to administer and manage your Adobe product licenses and users" — hierarchy: **IMS org → product (Adobe Experience Manager as a Cloud Service) → product profiles → users/groups**. Access to anything AEM is granted by putting a user in a *profile*, never directly on the product.

## Identity types (pick once, migrate painfully)

- **Adobe ID** — personal, created at adobe.com, **Adobe-managed, anybody can create one**; the account belongs to the person, not the org.
- **Enterprise ID** — **created only by system administrators; the organization owns the account**, Adobe just hosts it.
- **Federated ID** — org owns *and* authenticates: **SAML2 SSO** against the corporate IdP; no Adobe-held password. Enterprise default; pairs with the SP-side story in [sso-authentication-handlers.md](../infrastructure-ops/sso-authentication-handlers.md) and the IMS credential layer in [adobe-ims-authentication-types.md](adobe-ims-authentication-types.md).

Admin roles: **System Administrator** (the org-wide role; verify via the ADMIN ROLE column showing "System" under Administrators) above product administrators and product-profile administrators.

## Cloud Manager product profiles (org/program-level, one set)

Under the AEMaaCS product in Admin Console; **a user needs at least one profile to enter Cloud Manager at all**. The onboarding set:

- **Business Owner** — "manage different programs"; the journey tells the sysadmin to **self-assign Business Owner first** (program creation is BO-only, [cloud-manager-program-types.md](cloud-manager-program-types.md)).
- **Deployment Manager** — "deploy code from your repositories to running AEM environments" (pipelines, variables, logs — the role most API 403s point at, [cloud-manager-api-cli-sdks.md](cloud-manager-api-cli-sdks.md)).
- **Developer** — "develop custom AEM applications and commit code"; also Developer Console access ([cloud-manager-environments.md](cloud-manager-environments.md)).
- (Full role matrix adds **Program Manager** and the read-only **User** role — permission-by-operation table in [cloud-manager-api-cli-sdks.md](cloud-manager-api-cli-sdks.md).)

Users routinely hold several profiles; these are **not** AEM runtime permissions.

## AEM product profiles (per environment × instance)

Each environment's author gets its own pair, created with the environment:

- **AEM Users [env]** — "normal users who perform everyday content authoring"; login access to that author.
- **AEM Administrators [env]** — "users responsible for advanced features"; admin rights on that instance.

Rules that bite:

- **Per-environment assignment**: membership in dev's profile grants nothing on stage/prod — "repeat these steps for all environments." (The usual "works on dev, can't log into stage" ticket — and the Repository-Browser-missing triage in [aemaacs-repository-inspection-by-tier.md](aemaacs-repository-inspection-by-tier.md).)
- **"Do not edit or delete the product profiles named AEM Administrators or AEM Users — it can break login to the AEM cloud instance."** Verbatim warning; scope changes belong in AEM groups, not in reshaping these profiles.
- **Every AEM product-profile member automatically gets read-only Cloud Manager access** (the "User" role) — so seeing Cloud Manager proves nothing about deploy rights, and an AEM Administrator still can't run a pipeline without a separate Deployment Manager grant.
- Navigation: Cloud Manager → **Manage Access** deep-links into the right Admin Console profile.

Inside AEM, profiles surface as IMS-synced groups to hang ACLs on (fine-grained permissioning stays in AEM itself — groups/ACLs, repoinit for service users per [repoinit-acls-on-apps-and-libs.md](repoinit-acls-on-apps-and-libs.md)).

## Mental model for exam scenarios

Three permission planes, three tools: **Admin Console** (who may log in where: identity + profile), **Cloud Manager roles** (who may build/deploy/configure programs), **AEM groups/ACLs** (what they may touch in content). Distractors love moving a task one plane over — "give the developer deploy rights in AEM" or "fix author permissions in Admin Console" are both wrong by plane.

Source: AEMaaCS onboarding journey pages, Oct 2026. Docs-based.
