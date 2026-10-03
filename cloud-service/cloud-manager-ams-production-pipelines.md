# Cloud Manager (AMS) production pipelines

Configuring the **production pipeline** in Cloud Manager for AMS / AEM 6.x —
the deploy-to-stage-then-production pipeline, as opposed to non-production
(code-quality / dev-deploy) pipelines. What the KPI thresholds *measure* is in
[[cloud-manager-ams-program-setup-kpis]]; this note is the pipeline's own
switches. AEMaaCS step order is in [[cloud-manager-pipeline-steps]]; the
AMS-vs-cloud platform split in [[ams-vs-aemaacs]].

---

## Prerequisites and roles

- Program setup complete, at least one environment, and the Git repo has
  **at least one branch** — an empty repo blocks pipeline creation (same
  no-commits gate as repo validation, [[cloud-manager-git-repositories]]).
- **Deployment Manager** creates and configures pipelines.
- **Business Owner, Project Manager or Deployment Manager** can approve
  deployments.

Setup lives in three tabs: **Configuration**, **Source Code**,
**Stage Testing**.

## Configuration tab

### Stage settings

- **Deployment trigger**: *Manual* or *On Git Changes* (every commit to the
  configured branch starts the pipeline; manual start stays available).
- **Important Metric Failures behavior** — what happens when a quality-gate
  "important" (non-critical) failure occurs:
  - **Ask every time** (default) — pipeline pauses for a human,
  - **Fail Immediately** — cancels the pipeline,
  - **Continue Immediately** — overrides automatically and proceeds.
  (Critical failures always stop the pipeline; this switch governs only the
  important tier.)
- **Approve after Stage Deployment** — move the human approval to right
  after the stage deploy, *before* stage testing runs, instead of after.
- **Skip Load Balancer changes** — don't touch the LB during deploy
  (AMS-infrastructure knob with no AEMaaCS counterpart).

### Dispatcher invalidation (per environment, stage and production alike)

Paths to flush or invalidate after deployment: **max 100 paths per
environment**, action per path = **Invalidate** (preferred) or **Flush** —
Flush is needed mainly for **HTML client libraries**, where a touched
`.stat` isn't enough and the file must actually be deleted
([[dispatcher-caching-and-invalidation]] for the mechanics).

### Production settings — the three go-live gates

- **Use Go Live Approval** — a BO/PM/DM must click approve between stage
  and production.
- **Scheduled** — pipeline halts after stage and waits for a scheduling
  decision: **Now** / **Date** (pick the deployment time) / **Stop
  Execution** (abort the production half).
- **Use CSE Oversight** — the deployment is started by a Customer Success
  Engineer: **Any CSE** (first available) or **My CSE** (the assigned one,
  with backup coverage). **AMS-only** — there is no CSE step in AEMaaCS
  pipelines; in an exam option list "CSE oversight" is a tell that the
  scenario is AMS.

The three are independent checkboxes, so a pipeline can require approval
*and* a schedule *and* a CSE.

## Source Code tab — full-stack vs web-tier

- **Full Stack Code** pipeline: backend + frontend + HTTPD/Dispatcher config
  in one deploy. Option **Ignore Web Tier Configuration** excludes the
  dispatcher part (auto-selected once a web-tier pipeline exists for the
  environment).
- **Web Tier Config** pipeline: **HTTPD/Dispatcher configuration only** —
  ship Apache/dispatcher changes without rebuilding and redeploying the
  application. **Code Location** points at the dispatcher folder (default
  `/`); set it precisely or unrelated application code is pulled into the
  artifact and Apache can fail to restart.
- **One full-stack and one web-tier pipeline max per environment**, and the
  web-tier responsibility lives in exactly one of them (creating the
  web-tier pipeline flips the full-stack one to ignore its dispatcher
  config).

## Stage Testing tab

Feeds the performance-test gate measured against the program KPIs
([[cloud-manager-ams-program-setup-kpis]]):

- **Sites** — traffic distribution weights across **Popular Live Pages /
  Other Live Pages / New Pages** (i.e. cache-friendly vs cache-busting mix).
- **Assets** — **Images vs PDFs** split slider, plus optional **custom
  asset** uploads (your own representative PDF/image files) for the
  repeated-upload test.

## Smart Build

Maven **build-cache** for pipelines (code-quality + dev/stage/prod
full-stack): only changed modules and their dependents rebuild; the first
run is always a full build (cold cache). Opt a module out with
`<maven.build.cache.enabled>false</maven.build.cache.enabled>` in its
`pom.xml`. Limitation: the cache trusts the **Maven dependency graph** —
anything a plugin reads outside that graph (generated sources, filesystem
side inputs) won't trigger a rebuild, so such modules must opt out or stay
on full builds. Switching back to full build is a pipeline edit, not a new
pipeline.

## Exam checklist

- Deployment Manager configures; BO/PM/DM approve; repo needs ≥ 1 branch.
- Important-metric failures: **Ask every time** (default) / Fail / Continue
  — governs the *important* tier only.
- Production gates: Go Live Approval, Scheduled (Now/Date/Stop), **CSE
  oversight (Any/My) = AMS-only tell**.
- Dispatcher invalidation: ≤ **100 paths**/environment, prefer Invalidate,
  Flush for HTML clientlibs.
- Web-tier pipeline = dispatcher-only deploys; one full-stack + one
  web-tier per environment, dispatcher config owned by exactly one.

## References
- [Production pipelines (Adobe docs, Cloud Manager for AMS)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-manager/content/using/pipelines/production-pipelines)

Docs-based (page read Oct 2026), not verified against a live AMS program.
