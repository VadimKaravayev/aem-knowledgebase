# Author-Tier Scaling: Multi-Author Instances (AEM 6.5)

When and how to scale the author tier, and why it doesn't work like Publish-tier scaling.

---

## When to go multi-author

Two capacity drivers (not availability drivers — see below):
1. **Number of parallel authors** — one instance handles a finite number of concurrent authoring sessions before it degrades.
2. **Types of actions authors perform** — bulk uploads, heavy workflows, complex component-laden pages cost far more than simple text edits; a resource-heavy authoring pattern can force multi-author even with a modest headcount.

Not capacity drivers, even though they sound infra-related: **dispatcher cache efficiency** and **website traffic** are Publish-tier concerns and don't affect author-tier sizing. **Fail-safeness / HA** is a separate decision — it drives a standby/cluster-for-availability setup, not capacity-for-load.

## The key difference from Publish scaling: it's about the persistence backend

Publish scaling is easy to reason about: multiple Publish instances, each an independent replica of the same content, fronted by Dispatcher + a load balancer round-robining requests — any instance can answer any request.

**Authoring doesn't work that way by default**, because it depends on which Oak **NodeStore** backs the repository:

- **TarMK (Segment Node Store)** — the standard/default for AEM 6.5 authoring. **Does not support multiple Author nodes sharing one repository** — there is no clustering. Each Author instance is its own independent repository with its own content. A "multi-author setup" on TarMK therefore means **partitioning** — different tenants/teams/content areas live on different, separate Author instances — not load-balancing identical requests across a pool. Authors are pointed at *their* instance directly; there's no LB making that routing decision for them.
- **MongoMK (Mongo Document Node Store)** — swaps Tar files for MongoDB as the persistence layer, which **does** allow multiple Author instances to share one logical repository. This is the actual clustered/HA author pattern: Adobe's recommended AEM 6.5 multi-author topology is a **MongoMK Author cluster of 2+ instances**, backed by 2+ MongoDB replica-set members, replicating out to a (typically TarMK) Publish farm. With MongoMK, a load balancer in front of the author cluster genuinely makes sense — any node can serve any author, so requests can be distributed.
- **AEMaaCS** — the author tier auto-scales, which implies a cloud-native equivalent of the shared-repo (MongoMK-like) model under the hood, managed by the platform — not something configured directly.

**So: "is there a load balancer between authors?" depends entirely on whether the deployment is TarMK-partitioned (no LB, routing is "which instance owns this content") or MongoMK-clustered (yes, LB makes sense, any node serves any author).**

## A specific MongoMK gotcha worth knowing

Replication agents expect a **single target node**. If a MongoMK author cluster has replication agents fanning out to multiple Publish instances that are themselves backed by Mongo, Author throws **duplicate document errors** on replication — a real trap when combining MongoMK on both author and publish tiers rather than the standard MongoMK-author / TarMK-publish split.

## TarMK vs. MongoMK, in one line

**TarMK is optimized for performance** (fast local Tar-file I/O, no network hop to a DB) — the default choice absent a clustering requirement. **MongoMK is optimized for scalability/HA** (shared state across nodes, automated recovery if an instance or an entire datacenter fails) — the choice when you specifically need multiple Author nodes to share one repository.

## Exam signal

"Multi-author instance" scenarios test whether you distinguish **capacity drivers** (parallel authors, heavy action types → B, E in the source question) from **availability drivers** (fail-safeness → standby/cluster, not scaling) and from **publish-tier drivers** (traffic, cache efficiency → irrelevant to author sizing). A follow-up about load-balancing authors tests whether you know that answer is conditional on the NodeStore — TarMK partitions, MongoMK clusters.

## References
- [Recommended Deployments — AEM 6.5 (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/implementing/deploying/deploying/recommended-deploys)
- [TarMK Author Tier or MongoMK Author Cluster (Adobe Experience League Community)](https://experienceleaguecommunities.adobe.com/adobe-experience-manager-sites-8/tarmk-author-tier-or-mongomk-author-cluster-38182)
- [TarMK vs. MongoMK: Choosing the Right Persistence Backend for AEM (Medium)](https://medium.com/@ucgorai/tarmk-vs-mongomk-choosing-the-right-persistence-backend-for-aem-3f50beca9e8c)
- Question 3 in this repo (`questions/question-3.md`) — the capacity-planning scenario this note was written from.
