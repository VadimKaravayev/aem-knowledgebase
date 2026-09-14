# Adobe Managed Services (AMS) vs. AEM as a Cloud Service (AEMaaCS)

What AMS actually is, and the specific architectural reasons it can't substitute for AEMaaCS in scaling-driven scenarios (the exam's favorite trap).

---

## What AMS is

**Adobe Managed Services (AMS)** is Adobe's **managed hosting** offering for "classic" AEM 6.x (6.5 / 6.5 LTS) — the traditional **Author / Publish / Dispatcher** topology, just operated by Adobe on your behalf instead of you running your own datacenter or infra team. It is *not* a different product from on-prem AEM 6.5 — it's the same architecture, hosted.

- **Topology:** standard multi-tier — one or more Author instances, one or more Publish instances, Dispatcher(s) in front (an Apache module) doing caching/load balancing, often in a **multi-legged** arrangement (one Dispatcher per Publish leg, hardware load balancer in front).
- **You still get server access** — you (or a partner) can log in and inspect files, tune OSGi configs, work with the repository directly. This is the opposite of AEMaaCS's SaaS model.
- **Cloud Manager can front AMS deployments** for CI/CD (build/test/deploy pipelines) — but Cloud Manager is a deployment automation tool, not a scaling mechanism. It has nothing to do with runtime elasticity. See [[shared-datastore-binaryless-replication]] for a related "config vs. infra" distinction, though that note is AEMaaCS-side.
- **Sizing is fixed but adjustable on request** — environments are provisioned at a given size; scaling up/down is a change request to Adobe, not something that happens automatically in response to load.
- **Updates** ship as service packs, applied on-demand/scheduled — not continuous.
- **Licensing** is packaged (by number of environments, environment size, SLA tier) rather than usage-based.

### Where it physically runs

AMS is not a competitor to AWS/Azure — it runs **on top of** them. Adobe provisions VMs in **AWS or Azure (customer's choice)** and installs/configures AEM 6.5 on top, then operates it end-to-end: Adobe holds the only backend/SSH access, monitors it (Splunk, New Relic), and handles issues. So the customer never logs into the cloud console to manage servers directly — Adobe is the sole operator, the cloud provider is just where the VMs physically live. (VPC/VNet peering or Direct Connect back to the customer's own AWS/Azure account or on-prem network is available for integration.)

The architecture is unaffected by which cloud sits underneath — it's still the fixed-size Author/Publish/Dispatcher topology described above. This is a different thing entirely from AEMaaCS, which isn't "AEM installed on a VM" — it's a purpose-built SaaS platform, which is *why* only AEMaaCS auto-scales.

## What AEMaaCS is

A genuinely different deployment model — **SaaS**, cloud-native:

- **Auto-scaling** — both author and publish compute tiers scale elastically based on load (traffic spikes, editor load spikes), with no manual provisioning step.
- **No server access** — fully managed; everything goes through Cloud Manager (which here *does* also gate deployments, but the platform itself handles runtime scaling separately).
- **Continuous updates** — the platform is kept current automatically; you don't apply discrete service packs.
- **Combined CDN** (Fastly + Adobe CDN) is part of the platform for global delivery — see [[dispatcher-ignoreurlparams]] for how Dispatcher-tier caching concepts map (loosely) onto this CDN tier.
- **Usage-based licensing** — billed on content requests per month, not environment count/size.
- **Shared datastore is the managed default** — see [[shared-datastore-binaryless-replication]]; you don't hand-roll this on AEMaaCS the way you would on AMS/on-prem.

## The exam trap this feeds

A scenario states requirements like "scale for traffic spikes" and/or "scale for periods of higher editor load," then offers an option that *sounds* cloud-native — "AMS with Cloud Manager," "AMS with auto scaling" — banking on the reader pattern-matching "Managed Services" + "Cloud Manager" + "auto scaling" as buzzwords to a correct-sounding cloud answer. **AMS structurally cannot deliver elastic scaling** — environments are fixed-size, manually resized on request. Only AEMaaCS auto-scales both compute tiers. If a scenario's core requirement is elasticity (traffic spikes, editor-load spikes, unpredictable growth), the answer has to be AEMaaCS-based, full stop — no AMS configuration, however dressed up, satisfies it.

Mental model: **Cloud Manager = CI/CD** (works with either AMS or AEMaaCS, orthogonal to scaling), **auto-scaling compute = AEMaaCS-only**, **CDN = delivery-side spike absorption** (part of AEMaaCS's platform; AMS Dispatcher caching is a much more limited analog and doesn't scale compute).

## References
- [AEM Cloud Service vs Managed Services (Adobe Experience League Community)](https://experienceleaguecommunities.adobe.com/t5/adobe-experience-manager/aem-cloud-service-vs-managed-services/m-p/409582)
- [Introduction to the Architecture of AEM as a Cloud Service (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/overview/architecture)
- [Author and Publish Architectural Overview (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-screens/user-guide/administering/author-publish/author-publish-architecture-overview)
- [Secure digital experiences with AEM on AWS European Sovereign Cloud (Adobe)](https://business.adobe.com/uk/blog/the-latest/secure-digital-experiences-with-adobe-experience-manager-on-aws-european-sovereign-cloud)
- [Adobe Managed Services vs. Self-Hosting AEM — Pros & Cons](https://www.opsinventor.com/adobe-managed-services-vs-self-hosting-aem-pros-cons/)
- Question 26 in this repo (`questions/question-26.md`) — the retail-assets scenario this note was written from.
