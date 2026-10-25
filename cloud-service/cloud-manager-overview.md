# Cloud Manager — one-page overview and map of the detailed docs

Cloud Manager is Adobe's hosted CI/CD and environment-management console for AEM. It is **mandatory for AEM as a Cloud Service (AEMaaCS)** and optional-but-standard for Adobe Managed Services (AMS, hosted AEM 6.5). This page is the orientation layer; every section links to the doc that holds the verified detail.

## Hierarchy

IMS tenant → **program** → environments, Git repositories, pipelines.

- **One production environment per program**, so independently released sites need separate programs.
- Programs are **production** or **sandbox**. Sandboxes carry no live traffic, no auto-scaling, no custom domains or SLA; they **hibernate after 8h idle and are deleted after 3 months hibernated** (environments only — program, repo and code survive).
- Detail: [[cloud-manager-program-types]], AMS variant [[cloud-manager-ams-program-setup-kpis]], [[cloud-manager-ams-introduction]].

## Environments

- **Production + Stage exist only as a pair** and cannot be deleted in a production program.
- **Development** environments take non-production pipelines only.
- **Rapid Development Environment (RDE)** has no pipeline at all — code is synced with the `aio aem rde` CLI.
- **Name and primary region are immutable** once created; up to 3 additional publish regions.
- Sandbox/dev AEM updates are two steps: bump the pipeline's AEM version, then run it.
- Detail: [[cloud-manager-environments]], architecture context [[aemaacs-architecture-overview]], monitoring on AMS [[cloud-manager-ams-environment-monitoring]].

## Pipelines

- **Production pipeline**: Validate → Build → Unit Tests → Build Images → Deploy to Stage → Stage tests (performance, security, Experience Audit) → approval (manual or automatic) → Deploy to Production.
- **Non-production pipeline**: deploys to a dev environment, or runs code quality only.
- **Config pipeline**: ships CDN rules, WAF, traffic filters and similar configuration separately from code.
- **Build Images boots a transient AEM** to run repoinit and build indexes, so a repoinit failure aborts there, not at Deploy.
- **Performance and security tests run on Stage only**: a pipeline can block production with no code error at all.
- Detail: [[cloud-manager-pipeline-steps]], [[cloud-manager-ams-production-pipelines]], [[cloud-manager-experience-audit]], [[cloud-manager-faq-build-deploy]], build container [[cloud-manager-ams-build-environment]], project layout [[cloud-manager-ams-project-setup]], [[aemaacs-content-package-structure]], rolling deploys [[rolling-deployment-two-version-overlap]].

## Git repositories

Three ways a pipeline gets code:

1. **Adobe-hosted repository** (`git.cloudmanager.adobe.com`, Git-specific credentials from Access Repo Info).
2. **Private repository / bring-your-own GitHub** — public github.com only; pipelines read it directly and report PR checks.
3. **External repository** — any HTTPS Git host (GitHub Enterprise, GitLab, Bitbucket, Azure DevOps) via PAT + webhook.

Detail: [[cloud-manager-git-repositories]], multi-team aggregation [[cloud-manager-multi-repo-team-setup]].

## Roles

| Role | Can |
|---|---|
| Business Owner | create/edit/delete programs and environments, set regions, edit program entitlements |
| Deployment Manager | configure pipelines, add/validate repos, manage environment variables, run pipelines |
| Developer | push code, open Developer Console, read repo credentials |
| Program Manager | approve deployments, edit program settings |

Roles are assigned through **Admin Console product profiles**, never directly on the product. Detail: [[admin-console-product-profiles]].

## Supporting services

- **Custom domains and TLS**: Adobe-managed DV certificates (Let's Encrypt, auto-renew) or customer-uploaded certs — [[cloud-manager-ssl-certificates]], [[cloud-manager-custom-domain-names]].
- **IP allow lists and advanced networking** (VPN, dedicated egress, flexible port egress) — [[cloud-manager-environments]].
- **Variables and secrets**: build-time env vars set by CM (`CM_BUILD`), pipeline variables, and environment variables/secrets exposed as OSGi config placeholders — [[cloud-manager-variables]].
- **API, CLI and SDKs**: HAL HTTP API, `aio cloudmanager` CLI plugin, Java/JS SDKs — [[cloud-manager-api-cli-sdks]]; AMS New Relic user access is managed via that API — [[cloud-manager-newrelic-users-api]].
- **Actions Center** alerts and the code-quality analyzers, runnable locally with the AEM Analyser Maven plugin — [[aem-analyser-maven-plugin]], [[guava-deprecation-cleanup]].
- Platform comparison for 6.5/AMS versus AEMaaCS decisions — [[ams-vs-aemaacs]].

## Exam / design traps in one list

- One production env per program; Stage cannot exist without Production.
- RDE has no pipeline; dev environments cannot take a production pipeline.
- Build Images, not Deploy, is where repoinit breaks.
- Stage-only tests (performance, security, Lighthouse) can block a prod release with correct code.
- Bring-your-own GitHub is public github.com only; GHE is an *external* repository.
- Sandbox hibernation deletes environments after 3 months but keeps the program and repo.
- Environment name and primary region cannot be changed after creation.
