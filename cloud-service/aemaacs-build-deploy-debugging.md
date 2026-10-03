# Debugging AEMaaCS build and deployment failures

Adobe's per-step failure catalog for Cloud Manager pipelines (Experience
League tutorial), organized as "step → error signature → fix". Step order
and the Build Images transient-AEM mechanics in
[[cloud-manager-pipeline-steps]]; the FAQ's deploy triage in
[[cloud-manager-faq-build-deploy]]; blocked-queue internals in
[[content-distribution-journal]].

---

## Validation step

Config-level failures, no code involved:

- *"The environment is in an invalid state"* — target env mid-transition;
  wait (or recreate).
- *"The environment is marked as deleted"* — pipeline points at a deleted
  env; edit the pipeline, reselect.
- *"Invalid pipeline … Branch=xxxx not found in repository"* — branch gone;
  recreate it or repoint the pipeline.

## Build & unit test

Reproduce locally with `mvn clean package`. Usual causes: dependencies not
on Maven Central (private repo auth / unregistered repo in the pom),
**timing-sensitive unit tests built on `.sleep(…)`**, unsupported Maven
plugins.

## Code scanning

Static analysis ([[cloud-manager-code-quality-testing]]); critical security
findings fail, lesser ones are overridable; **Download Details** gives the
CSV of findings.

## Build Images

- **Duplicate OSGi configurations** — *"Unable to convert content-package …
  Configuration 'X' already defined"*: the same config PID arrives from two
  code packages (or one package embedded twice in `all`). Fix by
  deduplicating / resolving via run modes; check every non-container
  package carries `cloudManagerTarget=none`
  ([[aemaacs-content-package-structure]]). **The `mergeConfigurations`
  flag does not apply to AEMaaCS** — a 6.x answer that doesn't transfer.
- **Malformed repoinit** — can *pass silently on the local SDK yet fail at
  Build Images*; test by actually deploying the repoinit config to a local
  quickstart, not by eyeballing.
- **Repoinit content dependency unsatisfied** — the script assumes content
  that doesn't exist on a blank repo
  ([[repoinit-acls-on-apps-and-libs]]).
- **Core Components version mismatch** — *"Bundle X is importing package(s)
  com.adobe.cq.wcm.core.components… but no bundle is exporting these"*:
  the app upgraded Core Components but the **non-production environment
  hasn't taken the AEM release that provides them** (non-prod doesn't
  auto-update). Fix order: revert commit → update the environment's AEM →
  redeploy the upgrade; keep local SDK matched to the env's release
  ([[aemaacs-sdk]]).

## Deploy step

Logs: **Download Log** button, then the **`aemerror`** log for pod
startup/shutdown (not part of Cloud Manager's own download). Timestamps are
GMT.

- **Pipeline AEM older than the environment** — compare versions; update
  the env or **delete and recreate the pipeline** (the pipeline pins a
  release).
- **Cloud Manager timeout** — custom code doing heavy work (big queries,
  tree traversals) in bundle/component *lifecycle* during startup;
  `aemerror` shows what was running; move work off the activation path.
- **Incompatible code/config** — violations that only surface in the real
  container; grep `aemerror` for your packages' ERRORs.

### The `/var`-in-a-content-package signature (worth memorizing whole)

Symptoms line up as: first deployment *succeeds* but mutable content is
missing on Publish → activation/deactivation blocks on Author (distribution
queue, Tools → Deployment → Distribution) → **subsequent deployments fail
after a ~60-minute gap** between "Begin deployment" and "Failed deployment"
in the log. Cause: the replication/distribution importer can't write the
packaged `/var` content on Publish. Fixes, in Adobe's priority order:

1. drop `/var` from the package (usually unnecessary content),
2. create the structures via **repoinit** instead (run-mode-scoped to
   author/publish as needed),
3. author-only `/var` → a discrete package embedded under an
   **author run-mode install folder** in `all`,
4. last: ACLs for `sling-distribution-importer`
   ([[content-distribution-journal]]).

Escalation: Admin Console → Support tab → Create Case (check the org
switcher first).

## Exam checklist

- Validation = env/branch config errors, not code.
- Build Images: duplicate OSGi config = same PID from two packages
  (`mergeConfigurations` is not an AEMaaCS answer); repoinit can pass
  locally and fail here; "importing … but no bundle is exporting" on
  non-prod = env behind on AEM release, not a code bug.
- Deploy: `aemerror` log is the real source; startup-heavy custom code =
  timeout; **missing-on-publish + blocked queues + ~60-min deploy failure
  = `/var` in a content package** — fix with repoinit or run-mode-scoped
  packaging before reaching for importer ACLs.

## References
- [Debugging AEMaaCS: Build and Deployment (Experience League tutorial)](https://experienceleague.adobe.com/en/docs/experience-manager-learn/cloud-service/debugging/debugging-aem-as-a-cloud-service/build-and-deployment)

Docs-based (page read Oct 2026), failure signatures as documented.
