# Implementing an AEM connector for AEMaaCS: Adobe's rules for partner packages

Adobe's guidance for ISV/partner **connectors** (Exchange-listed integrations
such as translation, DAM, video, CRM, SEO plug-ins) that must install cleanly
on AEM as a Cloud Service. The useful content is not the pattern catalogue but
the **hard rules**: package split, `/apps/connectors/<vendor>`, no `/libs`,
no `/etc`, Cloud Services card writing to `/conf`, read config through
**Context-Aware Configuration**, and **Sling Scheduled Jobs** with idempotent
handlers for anything that pulls data.

Companion notes: [[all-package-embed-structure]] (how the `all` package
embeds the connector), [[rolling-deployment-two-version-overlap]] (why the
mutable/immutable split exists), [[httpclient-retries-timeouts-aem-connector]],
[[aem-sdk-java-ceiling-dual-65-cloud]] (one connector branch for 6.5 + Cloud),
[[translation-connector-cached-services]] and [[translation-rules-xml]]
(translation connectors specifically), [[aemaacs-sdk]] (local testing).

---

## 1. Integration patterns Adobe expects to see

| Pattern | Example | Implementation note from Adobe |
|---|---|---|
| **Pull** external data into AEM | CRM contacts shown on the site | **Sling Scheduled Jobs**, not schedulers or threads: jobs survive container loss and **may fire more than once**, so the handler must be idempotent |
| **Push** AEM data out | newsletter form → CRM | usually a Sling job or workflow step triggered by the submission |
| External system **reads assets** | PIM or other CMS linking to DAM renditions | serve via publish/CDN URLs or Assets APIs, not author |
| External system **stores assets** | MRM drops approved assets into DAM | upload APIs; mind the asynchronous processing model |
| Custom **UI component** | author drags a video component and picks a video | standard component + dialog; keep the vendor SDK out of the page render path |
| **Act on an asset** with a partner service | send asset to a video platform on publish | event/workflow driven, asynchronous |
| **Analyse** page/asset in the console | SEO recommendations | Touch UI extension, read-only against the repository |
| Page-level **user data** | demographics for personalisation | **ContextHub** is the framework; keep PII on publish out of the repository |
| **Translation** | copy and metadata | use the **AEM Translation Framework**; Adobe's Bootstrap Connector is the reference implementation |

Developer licences for building against AEM come through the **Adobe
Exchange Program**; the Partner Team also gives ISVs a **sandbox** to deploy
into a vanilla application before submission.

## 2. Package structure rules (non-negotiable on AEMaaCS)

- **Strict split** between immutable and mutable content, so rolling
  deployments work: code and configs in **`/apps`**, everything editable in
  **`/content` and `/conf`**. Follow the AEM Project Structure guidance;
  **existing connectors must be refactored** to comply, not grandfathered.
- **Only Adobe writes to `/libs`.** Partners and customers write to `/apps`.
- Anything a 6.x connector kept under **`/etc`** (configs, designs, cloud
  service configs) moves to `/conf` or another top-level folder, per the 6.5
  repository restructuring.
- Put the code under **`/apps/connectors/<vendor>`** so customers running
  several connectors get a clean tree.

## 3. Cloud Services configuration: the card under Tools → Cloud Services

The connector ships code that makes a **card with the connector's name**
appear under **Tools → Operations → Cloud Services**. Clicking it opens the
configuration browser, the customer picks a **parent folder**, and the
connector's form collects every property it needs, storing the result in a
**configuration folder under `/conf`**. That folder is what the customer
later selects on the **Sites** or **Assets** properties tab ("Cloud
Configuration").

Design consequences:

- The form is the connector's public configuration contract; secrets in it
  should be encrypted (CryptoSupport) or referenced via OSGi secret
  variables, never plain properties.
- Because the config is per `/conf` folder, one tenant can hold several
  independent connector configurations (multi-brand, multi-vendor-account).

## 4. Read configuration through Context-Aware Configuration, not a node path

Context-Aware Configuration layers config across `/libs`, `/apps`, `/conf`
and nested `/conf` folders with **inheritance**: a global setting with
per-microsite overrides. Cloud Services configurations can participate, so
connector code must resolve config via the **Context-Aware Configuration
API** (from the content resource) rather than hard-coding a configuration
node path.

If the connector **ships default configurations** that customers may modify,
architect for merging future defaults into customer-changed copies. Adobe's
warning: changing customer-customised content or configuration **without
notice and consent** breaks their site and your reputation.

## 5. Coding and testing

- AEMaaCS development guidelines apply in full: no local disk state, no
  long-running request-thread work, no reliance on a single node, respect
  the rolling two-version overlap ([[rolling-deployment-two-version-overlap]]),
  compile against `aem-sdk-api` only.
- Build and iterate on a **local SDK** ([[aemaacs-sdk]]), then prove it on
  the **Partner Team sandbox** against a vanilla application. If the same
  artifact must serve 6.5 customers, see
  [[aem-sdk-java-ceiling-dual-65-cloud]] before choosing the SDK version.
- ACS AEM Samples is Adobe's pointer for well-commented reference code.

## Exam and review checklist

- Scheduled pull integrations on AEMaaCS → **Sling Scheduled Jobs**,
  idempotent, never `@Scheduler`-style in-memory timers.
- Connector packages: `/apps` (immutable) vs `/content` + `/conf` (mutable);
  **`/apps/connectors/<vendor>`**; **never `/libs`**, **nothing in `/etc`**.
- Connector configuration surfaces as a **Cloud Services card** and lands in
  **`/conf`**, selectable on Sites/Assets folder properties.
- Code reads it via **Context-Aware Configuration** so inheritance and
  per-site overrides work.
- Shipping defaults? Plan the merge; never overwrite customer edits silently.
- Translation connectors build on the **Translation Framework**.
- Dev licence and test sandbox come from the **Adobe Exchange Program**.

## References
- [Implementing an AEM connector (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/connectors/implement)
- [AEM project structure (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/developing/aem-project-content-package-structure)
- [AEM as a Cloud Service development guidelines (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/developing/development-guidelines)
- [Context-Aware Configuration (Apache Sling)](https://sling.apache.org/documentation/bundles/context-aware-configuration/context-aware-configuration.html)
- [AEM Translation Framework Bootstrap Connector (GitHub)](https://github.com/Adobe-Marketing-Cloud/aem-translation-framework-bootstrap-connector)
