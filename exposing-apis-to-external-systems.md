# Exposing AEM APIs to external/third-party systems: never Author

When a requirement says "an external/third-party system needs to pull content or assets from AEM via an API," the first architectural decision is **which tier** answers the call — Author or Publish — and the answer is always **Publish** (or a purpose-built external-facing product like Assets [brand-portal.md](brand-portal.md), never Author/Publisher directly for a mass external audience).

## Why Author is off the table

- Author is the **editorial/writable** system: unpublished content, workflows, replication agents, package manager, CRXDE, OSGi console — a much larger and more sensitive attack surface than Publish.
- Author is typically kept off the public internet entirely (VPN / internal network only) in a well-run AEM environment. Exposing an API endpoint on Author to an external system means exposing Author itself to the internet, or punching a hole through to it — both are the actual security risk, independent of which auth scheme protects the endpoint.
- This holds even for AEM as a Cloud Service, where Author has no public-facing ingress by default.

## The correct pattern

1. **Replicate the content/assets to Publish** (normal activation/replication).
2. Expose the API on **Publish**, behind the **Dispatcher**, using whatever auth mechanism fits the constraints (e.g., OAuth server + JWT tokens when credentials can't be shared and client IPs are dynamic — see [content-services-vs-cf-headless-delivery.md](content-services-vs-cf-headless-delivery.md) for the JSON/GraphQL delivery paths themselves).
3. If the "third party" is actually an **external partner/agency needing curated, isolated, self-service access** (not just an API client), the Adobe-sanctioned product for that is **Assets Brand Portal** ([brand-portal.md](brand-portal.md)), which is explicitly designed so external users never touch Author or Publisher.

## Exam trap

A question may focus entirely on picking the right **authentication mechanism** (OAuth+JWT vs SAML vs Basic Auth vs IP whitelisting) for a scenario that says third parties connect to "the Author instance," and expect you to reason only about the auth layer. In a real design review, the more fundamental red flag — third parties should never be pointed at Author to begin with — should be raised first; the auth-mechanism question is really "given Publish is the right tier, which auth scheme fits these constraints."

Confirmed via AEM Architect exam question-35 discussion, Sept 2026.
