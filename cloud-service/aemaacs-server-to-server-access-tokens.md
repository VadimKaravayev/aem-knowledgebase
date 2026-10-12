# Access tokens for calling AEMaaCS Author from a non-AEM server

How a script, CI job or external service authenticates **against an AEM as a
Cloud Service Author instance** (not against Adobe's own APIs). The author
host accepts an **IMS bearer token**; the token comes from a **technical
account that AEM itself provisions** in the Developer Console's
*Integrations* tab. This is a different object from a Developer Console
*project* credential, and the fact that bites: it is **still
certificate + private-key based** (a signed JWT exchanged at IMS), even though
Developer Console project JWT credentials are end-of-life, see
[[adobe-ims-authentication-types]].

Companion notes: [[adobe-ims-authentication-types]] (Adobe-API side),
[[exposing-apis-to-external-systems]] (why external callers normally target
Publish), [[admin-console-product-profiles]] (how the technical account gets
permissions).

---

## 1. Two flows

| | Server-to-server | Developer (local) |
|---|---|---|
| Who | a non-AEM server/CI calling Author | a developer without IMS admin rights |
| Credential | service-credentials JSON (cert, private key, `clientId`, `clientSecret`) | **Local Development Token** button in Developer Console → Integrations → *Local token* |
| Token lifetime | ~**24 h**, mint via JWT exchange | max **24 h**, then click again |
| Credential lifetime | **1 year** by default, extendable | n/a |
| Identity on Author | the technical account's user, authorised via a product-profile group | the developer's own IMS user |

A local SDK Quickstart needs none of this: basic auth works there.

## 2. Server-to-server, step by step

1. **Create the technical account.** An **IMS org administrator** who is also
   in the *AEM Users* or *AEM Administrators* product profile on Author opens
   the environment's Developer Console → Integrations → **Create new technical
   account**. Without the IMS org admin role the console only shows a message
   saying that role is required. Limit: **10 technical accounts per
   environment, deleted ones included**. Note the environment is
   typically a dev environment, not production.
2. **Download the service credentials JSON** (any user with admin on the
   environment can retrieve it once created). Treat it as a secret; with
   several credentials, keep track of which JSON belongs to which call.
3. **Exchange for a token.** Sign a JWT with the private key and exchange it
   at IMS. Adobe's reference code is
   [adobe/aemcs-api-client-lib](https://github.com/adobe/aemcs-api-client-lib);
   Adobe says explicitly it is **sample code, not a maintained production
   dependency**, so copy and adapt rather than depend on it:

   ```js
   const fs = require('fs');
   const exchange = require("./your-local-aemcs-client");
   const config = JSON.parse(fs.readFileSync("aemcs-service-credentials.json", 'utf8'));
   exchange(config).then(accessToken => console.log(JSON.stringify(accessToken, null, 2)));
   ```

   The token lives ~24 h. Cache it for its lifetime; do not mint per request.
4. **Call Author** with a bearer header:

   ```bash
   curl -H "Authorization: Bearer <access_token>" \
     https://author-pXXXXX-eXXXXX.adobeaemcloud.com/content/dam.json
   ```

5. **Give it permissions.** The technical account is a user with no rights
   until it is in a group:
   - Admin Console → create an **AEM as a Cloud Service product profile** for
     the environment and add the technical account to it.
   - Wait **~10 minutes** for the profile to sync to Author.
   - Confirm the group/user appears under Tools → Security → Groups (or
     Users), then grant JCR permissions to that group under Settings →
     Security → Permissions.
   - Accounts created before mid-2023 (single-credential era) were never put
     in a product profile; Adobe recommends retrofitting one for consistency.

## 3. Rotation and revocation

- **Expiry (1 year).** Integrations → Technical Accounts → **Add certificate**
  extends by another year. Install the new JSON, verify a call works, then
  remove the old one, and make sure token generation now reads the new file.
- **Compromised key.** On the technical account click **Add new private key**
  (it becomes *current*), install and verify, then **revoke** the old
  certificate and confirm, then delete it.
- **Gotcha: certificates under one private key are one credential.** If any
  certificate on a private key is active, they all count as active. Revoking
  one certificate therefore does not isolate anything. For a real
  compromise, create a **separate private key** and revoke **every**
  certificate on the old one.

## 4. Quick answers

- External system must call **Author** (not Publish) → technical account +
  IMS bearer token; Publish-facing APIs go through Dispatcher instead,
  see [[exposing-apis-to-external-systems]].
- "I added the account but every call is 403" → it is not in a product
  profile group yet, or the 10-minute sync has not landed, or the group has
  no JCR permissions.
- "Token stopped working after a day" → expected; tokens are 24 h, re-exchange.
- "Everything stopped working after a year" → certificate expired; *Add
  certificate*.
- Developer testing against a cloud dev env → **Local Development Token**,
  24 h, no admin rights needed.

## References
- [Generating Access Tokens for Server-Side APIs (AEMaaCS)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/developing/generating-access-tokens-for-server-side-apis)
- [adobe/aemcs-api-client-lib](https://github.com/adobe/aemcs-api-client-lib) — JWT exchange sample code
