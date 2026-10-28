# Edge Delivery Services (EDS) overview

What EDS is, what it replaces, and how it sits next to a classic AEMaaCS
Java-stack site. Condensed from Adobe's overview page (last updated
2025-11-06); the deep documentation lives on **aem.live**, not Experience
League.

Companion notes: [[aemaacs-architecture-overview]] (the service view with EDS
Publish as a second origin), [[cloud-manager-program-types]] (EDS is a
solution on a Sites program), [[edge-functions]] (the only server-side seam
on an EDS site), [[aemaacs-licensing-content-requests]] (EDS counts content
requests too; HEAD-only health checks),
[[edge-delivery-media-bus]] (code/content/media buses; where images live),
[[edge-delivery-blocks-and-widgets]] (table → block markup, loader, block vs widget),
[[edge-delivery-project-anatomy]] (repo layout, special files, Configuration Service),
[[edge-delivery-block-collection]] (boilerplate / collection / Block Party blocks),
[[edge-delivery-dev-good-practices]] (PR, CSS, JS and content-first rules),
[[edge-delivery-configuration-service]] (site config via Admin API, repoless, safe changes),
[[edge-delivery-translation-localization]] (multilingual sites, DA Translate app, placeholders).

---

## 1. What it is

A content delivery framework **inside AEM**, not a separate product: rendering
and delivery move to the network edge. It **replaces AEM Publish + Dispatcher
and the Core Components rendering stack** with a multi-cloud SaaS backend
plus plain front-end code. It is **not a CDN replacement**: it sits behind
your own CDN or the included [Adobe-managed CDN](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/content-delivery/cdn).

**How it works.** Front-end code (HTML, modern CSS, vanilla JavaScript; no
transpiler, bundler or build config) lives in a **GitHub repository**, which
is also the deployment mechanism. Content is fetched from the content source
on request; **new content needs no rebuild/redeploy**. Adobe's pitch is that
every developer can hit a **100 Lighthouse score** and that sites are tuned
for SEO and for generative-engine discovery (GEO).

## 2. Two authoring models, one delivery

| Model | Where content lives | Who it suits | Notes |
|---|---|---|---|
| **In-context editing** | AEM Author, edited in the **Universal Editor** (WYSIWYG, live preview) | teams that need AEM workflows, governance, permissions | Extension points + CSS/JS customisation via the GitHub repo. Doc: [Authoring with AEM for EDS](https://www.aem.live/docs/aem-authoring). |
| **Document-based** | Microsoft Word / Excel in **SharePoint**, or **Google Drive** for shops without SharePoint | marketing teams who already write in documents | No AEM Author in the loop. Doc: [Document-based authoring](https://www.aem.live/docs/authoring). |

Both feed the same EDS delivery. Content can also come from **other
systems**, including an existing AEM instance, so EDS is a migration-friendly
front door rather than a rip-and-replace.

## 3. Relationship to AEM Sites

- EDS and classic AEM Sites can share **one domain**; each can consume
  content from the other. The usual pattern is high-traffic marketing/landing
  pages on EDS, transactional or deeply componentised pages on the Java stack.
- It works with **Adobe Target** ([integration](https://www.aem.live/developer/target-integration))
  and **Experience Platform Tags/Launch**.
- Headless and headful delivery are both supported ("composable").
- Cloud Manager side: [Introduction to EDS in Cloud Manager](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/edge-delivery-sites/introduction-to-edge-delivery-services)
  and [Add an Edge Delivery site](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/edge-delivery-sites/add-edge-delivery-site).
  **Register the site in Cloud Manager**; Adobe support will not act on an
  unregistered site.

## 4. Operational Telemetry

EDS collects operational data to find functional and performance issues.
It is **sampled** and **excludes PII** by design. Doc:
[Operational Telemetry](https://www.aem.live/docs/operational-telemetry).
Useful answer for privacy questionnaires.

## 5. Getting help (three layers)

1. **Community**: Experience League Community, and the
   [aem-live Discord](https://discord.gg/aem-live), which is where Adobe
   engineers actually answer EDS questions.
2. **Product collaboration channel** for specific product questions
   (customer-specific, set up with Adobe).
3. **Support ticket** for major/critical issues under the contractual SLA.
   Recipe: site registered in Cloud Manager → standard support process →
   put **"Edge Delivery" in the ticket title** → include the **live site URL
   and the `.hlx` origin URL**.

## 6. Quick answers

- "Does EDS replace our CDN?" → no; it replaces Publish/Dispatcher
  rendering and sits behind a CDN (yours or Adobe's).
- "Do we need a pipeline/build for content?" → no; content changes are live
  on publish, code deploys through GitHub.
- "Where's the server-side logic?" → there is none in EDS itself; see
  [[edge-functions]].
- "Can authors keep using AEM?" → yes, via Universal Editor; or go
  document-based with SharePoint/Google Drive.
- "Fastest way to try it" → [pre-built tutorial environment](https://www.aem.live/developer/ue-trial),
  or own setup in under 30 minutes via the
  [Universal Editor developer tutorial](https://www.aem.live/developer/ue-tutorial).

## References
- [Edge Delivery Services Overview (AEMaaCS)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/edge-delivery/overview)
- [aem.live documentation hub](https://www.aem.live)
- [Universal Editor introduction](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/developing/universal-editor/introduction)
