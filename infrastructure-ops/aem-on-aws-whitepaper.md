# Running AEM on AWS (AWS whitepaper) — self-hosted IaaS patterns

[PDF](https://d0.awsstatic.com/whitepapers/Adobe%20AEM%20on%20AWS.pdf), AWS + Razorfish, **July 2016, written for AEM 6.1/6.2** — so instance types (M3/M4) and version talk are dated, but it remains the canonical writeup of the **third deployment model**: self-hosted AEM on IaaS, vs AMS (Adobe-operated on the same clouds, [ams-vs-aemaacs.md](../cloud-service/ams-vs-aemaacs.md)) vs AEMaaCS. Its patterns are what AEMaaCS later productized — reading it explains *why* the cloud service is shaped the way it is.

## Topology and network

Standard three tiers (author / publish / dispatcher — "author and publish are identical software, differentiated by configuration only"). Two VPC layouts:

- **All-public subnets** — simplest; every component gets its own security group; publish spread across ≥ 2 AZs.
- **Public + private** (recommended direction): web server + dispatcher public, **author and publish in private subnets** — but then authors need **VPN** access (or the alternative: an **author dispatcher** exposed only to content admins), and **NAT gateway/instances** are required for *outbound* traffic — AEM's Link Checker and update/patch retrieval silently need internet egress people forget about.

Sizing: general-purpose (memory+CPU+I/O balance) for author/publish; **the dispatcher tier is I/O-critical** — and all AEM EBS volumes should be GP2 or provisioned-IOPS.

## Load balancing and sessions — the durable mechanics

ELB fronts the **dispatchers**; enable **cross-zone load balancing** or distribution is per-AZ, not per-instance. The session fact worth keeping: AEM login state = token under the user node's **`.tokens`** node, mirrored in the **`login-token` cookie** → classic setup needs **sticky sessions** routing that cookie to one instance; **6.1+ can validate the auth cookie across all publish instances**, but only if application session state (e.g. a cart) is also shared — authentication ≠ session data ([scalability.md](../system-design/scalability.md): "sticky sessions are a confession"). ELB + Certificate Manager does HTTPS offload and port 80→4503 translation, letting publish hide in private subnets.

## HA and scaling

- Per-instance **EC2 Auto Recovery**; Auto Scaling groups across AZs; **Route 53 health-check DNS failover** for multi-region.
- Separate **AMIs + launch configurations per tier** (author vs publish pools).
- **Publish bootstrap trick:** keep the AEM install on its own EBS volume, snapshot frequently, launch new publishers from the latest snapshot so they start near-current, then **trigger author→publish replication on startup** to close the gap — the hand-rolled ancestor of AEMaaCS's journal catch-up ([content-distribution-journal.md](../cloud-service/content-distribution-journal.md)).
- **Dispatcher autoscaling is easy (stateless) except for invalidation wiring**: a freshly launched dispatcher must be known to the flush agents. The paper's fix: **co-locate dispatcher with its publish instance 1:1 so cache invalidation is a localhost call** — exactly the pairing AEMaaCS bakes in ([aemaacs-architecture-overview.md](../cloud-service/aemaacs-architecture-overview.md)).
- **MongoMK makes author/publish autoscaling "more straightforward" because the repo lives in the database** — the 6.x answer to elastic author capacity ([multi-author-scaling.md](multi-author-scaling.md)).

## CDN (CloudFront) and the two invalidation strategies

CDN *in addition to* dispatcher caching. Either:

1. **Event-driven purge** — custom replication event listener / workflow step / dispatcher `/invalidateHandler` script calling the CloudFront invalidation API; **delay the purge until all publishers have the content** or the CDN re-caches the old page from a lagging publisher (the same race as the author-side flush agent trap in [dispatcher-caching-and-invalidation.md](../dispatcher-url-delivery/dispatcher-caching-and-invalidation.md));
2. **TTL-driven** — CloudFront `minTTL=0` + `s-maxage`/`max-age` Cache-Control headers set in Apache: "CDN invalidations are effective every s-maxage seconds". Code deployments then need a full-distribution invalidation.

Dynamic fragments on cached pages: client-side JS, SSI, or ESI via **Varnish replacing the dispatcher** — the era's answer to what [sling-dynamic-include.md](../dispatcher-url-delivery/sling-dynamic-include.md) now does inside the dispatcher.

## Security

App layer: AEM + dispatcher security checklists, Apache `mod_rewrite` against DoS request flooding, `mod_security` against XSS. Infra layer: VPC subnet/SG/NACL layering, and the portable idea — **replicate your dispatcher `mod_security` rules into AWS WAF at CloudFront**, so attacks die at the edge before reaching origin (the posture AEMaaCS's CDN WAF-DDOS option now sells, [cloud-manager-program-types.md](../cloud-service/cloud-manager-program-types.md)). Plus CloudTrail/Config audit, Inspector/Trusted Advisor assessment.

## DAM, storage, backup

- Offload rendition work (FFmpeg/ImageMagick) to a **separate ASG of workers** — the self-hosted ancestor of Assets Compute.
- **S3 as data store, shared across author and even publish instances** — the whitepaper's version of [shared-datastore-binaryless-replication.md](shared-datastore-binaryless-replication.md); binary-less replication + lower storage/transfer.
- Backups: TarMK = **EBS snapshot with `fsfreeze`** to suspend FS I/O for consistency (or AEM online backup); MongoMK = back up the database; S3 datastore files are **immutable → incremental `aws s3 sync`/rsync works**, cross-region bucket copy = DR. Restore-testing rule from [monitoring-and-maintaining.md](monitoring-and-maintaining.md) applies unchanged.
- **EFS** as shared-FS alternative: Tar + EFS for faster HA instance creation, or EFS instead of S3 for the datastore.

Deployment automation: Jenkins as hub driving AEM's HTTP interfaces (curl — [aem-curl-operations.md](aem-curl-operations.md)), CloudFormation/OpsWorks for environment provisioning, CloudWatch custom-metric dashboards.

## Why keep a 2016 paper

Almost every pattern here is a hand-built version of something AEMaaCS later made a platform property: publish-paired dispatchers, snapshot-boot + replication catch-up vs journal replay, shared blob store, edge WAF, rendition offload. For design questions it's the reference for "customer insists on self-hosting AEM on AWS — what do they now own?": VPN/NAT plumbing, session stickiness, CDN invalidation races, snapshot-consistent backups, and all scaling choreography.

Source: AWS whitepaper "Running Adobe Experience Manager on AWS", July 2016, read in full Oct 2026.
