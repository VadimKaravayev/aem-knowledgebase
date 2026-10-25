# Dev, Stage and Production on AEMaaCS — what each tier is for and how they differ

Orientation page for the environment **tiers** of an AEM as a Cloud Service production program. The Cloud Manager mechanics (creation, roles, regions, updates) live in [[cloud-manager-environments]]; this page answers "which tier do I use for X, and why does it behave differently".

## The tiers side by side

| | **RDE** | **Development** | **Stage** | **Production** |
|---|---|---|---|---|
| Purpose | minutes-fast iteration on one developer's code | integration testing of a branch | production rehearsal + mandatory gates | live traffic |
| How code arrives | `aio aem rde install` (bundle/package/config/dispatcher), no pipeline | **non-production pipeline** only | **production pipeline**, first stop | production pipeline, after Stage + approval |
| Sizing | small, single-purpose | **undersized** vs prod | **same sizing as prod** (1:1) | full |
| Auto-scaling | no | no | yes (same as prod) | yes |
| Tests that run | none | code quality (+ optional functional/UI) | **performance, security, Experience Audit (Lighthouse), functional/UI** | none (post-deploy only) |
| Content | whatever you push; `reset` wipes | your test content | should mirror prod (Content Copy from prod) | real |
| Repository inspection | Developer Console | Developer Console | logs / Content Copy / Repository Browser — **no CRX/DE** | same as Stage |
| Lifecycle | can be reset in minutes | can be deleted / recreated by DM or BO | **exists only as a pair with prod**, undeletable | undeletable |
| AEM version | tied to pipeline version, updated by running it | same | auto-updated with prod | **auto-updated monthly** (maintenance releases, rolling) |

Sandbox programs have the same tiers in name but **no auto-scaling, no live traffic, no automatic AEM updates, hibernation after 8h idle** — never load-test on one ([[cloud-manager-program-types]]).

## The rules that follow from the table

- **Stage is the only place performance and security tests run.** A production pipeline can block a release on Stage with code that passed everywhere else; a dev environment never tells you about it ([[cloud-manager-pipeline-steps]], [[cloud-manager-experience-audit]]).
- **Dev is undersized on purpose.** Timings, memory behaviour and index build times on dev do not predict prod; use Stage (or the Specialized Testing Environment) for anything capacity-related.
- **Stage and Prod are one unit.** Same sizing, same regions (extra publish regions set on prod apply to stage), same auto-updated AEM version, created and deleted together.
- **RDE is not a dev environment.** No pipeline means no code-quality gate and no AEM version management by you; it is a scratchpad whose fidelity comes from running the real cloud runtime, unlike the local SDK ([[aemaacs-sdk]]).
- **Content flows down, code flows up.** Content Copy pulls prod/stage content into lower tiers; code only ever reaches a higher tier via a pipeline ([[rolling-deployment-two-version-overlap]]).
- **Every tier is immutable at `/apps` and `/libs`**; the difference between tiers is never "can I hot-fix in CRX" — it is which pipeline and which tests ([[aemaacs-repository-inspection-by-tier]], [[repoinit-acls-on-apps-and-libs]]).
- **Environment variables and secrets are per environment**, so the same build behaves differently per tier through OSGi placeholders, not through run modes you invent ([[cloud-manager-variables]], [[aemaacs-osgi-configuration]]).

## Run modes per tier

Cloud defines them for you: `prod`, `stage`, `dev` crossed with `author`, `publish`. OSGi config folders are `config.author`, `config.publish`, `config.dev`, `config.stage`, `config.prod`, `config.author.dev` and so on — no custom run-mode names. RDEs run with `dev`.

## Who uses which tier

| Role | Typical tier |
|---|---|
| Developer | RDE for the feature, Dev for the branch |
| QA | Dev (functional), Stage (regression, performance) |
| Author / content team | Stage for UAT with prod-like content; Prod |
| Release manager (Deployment Manager / Business Owner) | Stage gates and the prod approval |

## Exam / design traps

- "Run the load test on dev to save Stage time" — dev is undersized and un-scaled; results are meaningless.
- "Deploy the hotfix straight to prod with a non-production pipeline" — impossible; prod only takes the production pipeline via Stage.
- "Create a stage-only environment for UAT" — Stage exists only paired with prod; a second program is the only way to get another Stage.
- "RDE for the team's shared integration environment" — RDE has no pipeline and resets wipe it; that is what Dev is for.
- "Prod and Stage drifted to different AEM versions" — they cannot; only sandbox/dev versions are pipeline-driven.
