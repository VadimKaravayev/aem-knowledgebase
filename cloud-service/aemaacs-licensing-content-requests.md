# AEMaaCS licensing: Content Requests, entitlements, and the PSLT

Three layers people conflate: the **Product Description** ([helpx](https://helpx.adobe.com/legal/product-descriptions/adobe-experience-manager-cloud-service.html) — canonical but bot-blocked to fetchers; figures below from the edition **effective Sept 2, 2026**), the **PSLT** legal terms ([2025v1 PDF](https://www.adobe.com/cc-shared/assets/pdf/legal/terms/enterprise/pdfs/pslt-aem-cloud-service-ww-2025v1.pdf), read in full), and the **operational counting rules** ([Understand Content Requests](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/content-requests), current). Usage-based licensing is the AEMaaCS-vs-AMS economic axis ([ams-vs-aemaacs.md](ams-vs-aemaacs.md)); the Data Service exposes the metrics ([aemaacs-architecture-overview.md](aemaacs-architecture-overview.md)).

## The licensed metric: Content Requests

**"Counted either as a Page View or 5 API Calls, measured at the ingress of the first caching system to receive a Content Request."** So one HTML page view = 1 CR, and **5 JSON/API calls = 1 CR** — headless traffic is 5× cheaper per request. Quantity is licensed **monthly but compliance is tested over a rolling 12-month sum** — a Black-Friday month doesn't breach by itself; the year's total does.

**Sites plan includes** (Sept 2026 PD): 1× Production + Stage + Development + RDE, **1 TB storage each on prod and stage**, and author concurrency scoped by volume — **20 concurrent authors ≤ 5 M CRs/mo, 40 above**. Exceeding the licensed annual sum at any point = license more.

## Counting rules (the operational doc — exam-worthy exclusion list)

Measured **server-side at CDN ingress**, never client-side analytics (which undercount: consent walls, blockers, prefetch, API traffic — Adobe's own explanation for "analytics says less than the License Dashboard").

**Counts:** `text/html` and `application/json` responses with status **100–299**, including curl/wget/axios/headless-Chrome traffic, unknown bots, monitoring tools, and `<link rel="prefetch">`.

**Does NOT count:** 3xx/4xx/5xx responses; static assets (JS/CSS/images); **`/etc.clientlibs/*`**; `favicon.ico`; `/libs/` JSON (CSRF tokens); `/api/graphql` forwarded to Commerce (no double-count); **well-known search/AI crawlers** (Googlebot…, ChatGPT/Claudebot/Perplexity…); auto-blocked DDoS traffic; Adobe's own New Relic checks and `skyline-service-warmup` pod warm-ups; **Experience Fragments fetched same-domain** (referrer host matches, `/content/experience-fragments/...`).

Design consequences: route health checks to **`/system/probes/health`** (free) or use **HEAD not GET** (the only option on Edge Delivery, which lacks that probe path); on a **customer CDN, forward the real browser `User-Agent`** or the crawler exclusion can't fire and bots bill as humans.

**Customer-CDN audit regime** (PD): auditable measurement reports **every 6 months** (or within 30 days of request), counting **edge + origin**, client-side analytics inadmissible; no reports → Adobe may compute from **origin logs × assumed 95% cache rate** (an assumption that rarely flatters). Average payload **> 10 MB/request → license additional network capacity**. License Dashboard data refreshes every 24 h.

## PSLT 2025v1 — the terms worth knowing as an architect

- **Customer owns all testing** of Customer Customizations (quality *and* security); Adobe disclaims defects caused by them — the legal backdrop for the pipeline gates being non-negotiable.
- **Development Consultant** (SI) clause: the team must include **at least one person holding a current AEM developer certification**, "significantly engaged" — a cert requirement embedded in the license terms; Adobe may deny access to a consultant harming the service.
- **Data retention: 30 days after license end** for Customer Data *and code* — "the Cloud Service should not be Customer's only repository for Customer Customizations" (keep your own Git remote; see BYO-GitHub in [cloud-manager-git-repositories.md](cloud-manager-git-repositories.md)).
- **AEM SDK** is licensed for on-premise **dev/test/QA only** — never stage/prod ([aemaacs-sdk.md](aemaacs-sdk.md)).
- **App Builder is included** with the Cloud Service, subject to its own PSLT; customer-built extensions are Customer Customizations. (Note vs [app-builder.md](app-builder.md)'s "confirm entitlement" caution: base *use* rights ship with AEMaaCS per PSLT 2025v1 — what varies by contract is quotas/tiers, and 6.5/AMS has no such inclusion.)
- **Edge Delivery data regions are Adobe-discretionary** for published/preview content and transient form submissions — a compliance answer for data-residency questionnaires, and a real constraint for strict-residency programs.
- SLA = the **Unified SLA + Actionability Addendum** (adobe.com/legal/service-commitments); updates including Emergency Updates are Adobe-pushed — if an Update breaks Customer Customizations, Adobe only "supports your efforts" to fix your code.
- Automated Forms Conversion: don't feed it forms with **prefilled/personal data**.

Sources: PSLT 2025v1 (full read), Experience League content-requests doc, product description figures effective 2026-09-02 via licenseware.io wiki (helpx original 403s automated fetchers — re-verify plan numbers there before contractual use).
