# Sandbox program ("dev sandbox") — what it is, what it lacks, how it sleeps

The AEMaaCS program type for **training, demos, enablement, proofs of concept and documentation**, as opposed to a **production program** that serves live traffic. Orientation page; the full comparison table, hibernation mechanics and deletion flows are in [[cloud-manager-program-types]].

## What you get automatically

- **Sites, Assets and Edge Delivery Services** enabled — no solution mapping step.
- **One development environment** (author + publish). No stage, no prod, no RDE.
- A **Git repository seeded with the AEM Project Archetype sample project** on a branch ([[cloud-manager-git-repositories]]).
- A **non-production pipeline** already wired to build and deploy that branch ([[cloud-manager-pipeline-steps]]).
- Creation is a single naming step, not the production wizard.

## Restrictions, in full

| Missing | Consequence |
|---|---|
| Live traffic, AEMaaCS commitments, 99.99 % SLA | never point a real domain or audience at it |
| **Auto-scaling** | **performance and load tests are invalid** — use a production program's Stage ([[environment-tiers-dev-stage-prod]]) |
| Custom domains, IP allow lists, additional publish regions | cannot rehearse [[cloud-manager-custom-domain-names]] / [[cloud-manager-ssl-certificates]] here |
| Advanced networking (VPN, dedicated egress IP, non-standard ports) | integrations needing fixed egress cannot be proven on a sandbox |
| **Automatic AEM updates** | update = bump the pipeline's AEM version, then run it ([[cloud-manager-environments]]) |
| Technical support for issues *inside* the sandbox | support still covers creating/managing the program itself |

## Hibernation

- Hibernates after **8 hours** with no author, preview or publish requests. Visitors get a landing page.
- **Anyone with AEM product access** can de-hibernate it (Cloud Manager or Developer Console) — not just admins.
- **Pipelines and manual AEM updates still apply while asleep**; a deploy to a hibernated sandbox succeeds.
- After **3 continuous months** hibernated, Adobe **deletes the environments only**. The program, repository and code survive; recreate the environment and redeploy.

## Deletion

Single step by the **Business Owner**, effective immediately (environments and pipelines go). A Deployment Manager can instead delete just the environments. Contrast: a production program's two-phase mark → takedown window → permanent removal.

## Traps

- Load/performance test on a sandbox — invalid twice over (no auto-scaling, no commitments).
- Support ticket for a runtime problem inside the sandbox — declined by design.
- "The sandbox disappeared" — it hibernated and hit the 3-month deletion; the program and code are still there.
- Expecting a custom domain, allow list or VPN before the production program exists.
- Assuming AEM updated itself — on a sandbox nothing updates without a pipeline run.
