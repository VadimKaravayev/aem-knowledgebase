# Cloud Manager environments: types, roles, regions, preview service, updates

What you can do to an environment in Cloud Manager, who is allowed to do it,
and the rules that are easy to get wrong: the four environment types plus the
Specialized Testing Environment, what is immutable after creation, how
additional publish regions behave, why the preview service is unreachable out
of the box, and the two-step AEM version update. Companion to
[[cloud-manager-program-types]] (program level) and
[[aemaacs-architecture-overview]] (what the tiers are).

---

## Environment types

| Type | Sizing | Pipelines | Notes |
|---|---|---|---|
| Production + Stage | production sizing, both | production pipeline | **Created and managed only as a pair.** No prod-only or stage-only environment exists. Performance and security tests run on stage. |
| Development | smaller | **non-production pipelines only** | Not for performance or security tests. |
| Rapid Development (RDE) | dev-class | none, direct deploy via `aio` | For iterating on code already validated locally. |
| Specialized Testing Environment | near-production | dedicated | Stress testing and advanced pre-deployment checks under near-production conditions. Separate entitlement. |

Capabilities of any environment follow the solutions enabled on its program
(Sites, Assets, Forms, Screens). The number of environments you may create is
shown in parentheses next to each type in the Add dialog. If **Add Environment**
is greyed out, it is either missing permissions or exhausted entitlement.

## Who can do what

| Action | Role required |
|---|---|
| Add or edit an environment (including regions) | **Business Owner** |
| Update a dev or sandbox environment's AEM version | Deployment Manager or Business Owner |
| Delete a development environment | Deployment Manager or Business Owner |
| Open the Developer Console | Developer (any user with access, on a sandbox program) |

## Immutable after creation

- **Environment name.** Cannot be renamed. The description can.
- **Primary region.** Cannot be changed. Additional regions can be added and
  removed later.

## Additional publish regions

- Up to **three additional** publish regions beyond the primary, configured by a
  Business Owner on the **production** environment. The setting applies to
  **both production and stage**, and can only be edited from production.
- Add and remove are **separate transactions**. To swap one region for
  another: add, save, then remove, or the reverse.
- **Provision advanced networking before adding regions.** Otherwise the extra
  regions' egress traffic is routed through the primary region's proxy.
- Region health shows in the Status column of the Environments card and the
  Environment Segments table. Cloud Manager retries recovery on its own and
  traffic always routes to the closest **online** region. If a region stays
  unhealthy for several hours, remove and re-add it to force a full deployment,
  then contact Customer Care if it persists.
- The Cloud Manager API lists the currently available regions.

## Preview service: locked by default

Every environment gets a **preview service** delivered as an extra publish
service. On creation it carries a default IP Allow List named
`Preview Default [<envId>]` that **blocks all traffic**. The lock icon next to
the service name means this list is still applied. To open it:

1. Create your own IP Allow List and apply it to the preview service.
2. Unapply `Preview Default [<envId>]` in the same step, or use the allow-list
   update workflow to swap the default IP for yours.

Only then can authors publish to preview via Manage Publication. The
environment must be on AEM `2021.05.5368` or newer, so an update pipeline must
have run at least once.

## Updating the AEM version

Production programs are updated by Adobe automatically. **Sandbox** programs,
and historically development environments, show **Update Available** on the
Environments card when a newer public AEM release exists. Since 2024 most dev
instances and some sandboxes are also auto-updated, so the manual option may
be absent.

Because pipelines are the only deployment path, each pipeline is pinned to an
AEM version and the update is **two steps**:

1. **Update** the pipeline to the latest AEM version (from the environment's
   ellipsis menu).
2. **Run** the pipeline to deploy that version to the environment.

| State | What Update does |
|---|---|
| Pipeline already updated | prompts you to run it |
| Pipeline currently updating | tells you an update is running |
| No pipeline exists | prompts you to create one |

An update without a properly configured pipeline is not possible. Updating
either production or stage updates the other, so the pair stays on one release.

## Deleting

- Development environments: yes, by Deployment Manager or Business Owner.
- Production and stage in a **production** program: **cannot** be deleted.
- Production and stage in a **sandbox** program: can be deleted.

## Other environment-level actions

- **Manage Access** jumps to the author instance to manage users and groups.
  Access to the product itself is governed by team and product profiles in the
  Admin Console.
- **Developer Console** and **Local Login** are in the environment's ellipsis
  menu.
- **Custom domain names**: supported for Sites programs, on publish and
  preview. Not on author.
- **IP Allow Lists**: supported for Sites programs on author, publish and
  preview. Applying a list links all its IP ranges to that service.
- **Restore content** and **restore previous code** are separate self-service
  features linked from this page.

## Exam and review checklist

- "Create only a staging environment" or "delete stage but keep production" is
  impossible in a production program. They are a pair.
- Attaching a **production pipeline to a development environment** is
  invalid. Dev takes non-production pipelines only.
- Stress or near-production testing that dev cannot host and stage should not
  absorb is what the Specialized Testing Environment is for.
- A new preview URL returning nothing is the default allow list, not a
  publishing problem.
- Changing a region or renaming an environment means recreating it.
- An "Update Available" badge on a sandbox is cleared by updating the pipeline
  **and** running it. Updating alone deploys nothing.
- Traffic from an extra region unexpectedly leaving through the primary
  region's IP means advanced networking was provisioned after the region.

## References
- [Manage environments (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/manage-environments)
- [Additional publish regions (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/operations/additional-publish-regions)
- [Rapid Development Environments (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/developing/rde/overview)
- [Introduction to IP Allow Lists (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/ip-allow-lists/introduction)
