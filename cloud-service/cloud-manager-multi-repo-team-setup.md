# Multi-repo and multi-team setup for Cloud Manager

Adobe's pattern for **several customer-owned Git repositories** (teams,
vendors) feeding **one Cloud Manager repository**, and the enterprise
branching/governance model around it (AEMaaCS doc set). How a pipeline gets
wired to a single repo (Adobe-hosted vs BYO GitHub vs external) is
[[cloud-manager-git-repositories]]; triggering pipelines from scripts is
[[aio-cloudmanager-cli]] / [[cloud-manager-api-cli-sdks]].

---

## The aggregation pattern

Teams never develop in the Adobe repo directly. Each team/vendor keeps its
**own repository** (started from the AEM Project Archetype), and CI
automation (GitHub Actions or Jenkins) **pushes each repo's contents into a
dedicated subdirectory of the Cloud Manager repo**:

```
cloud-manager repo:
  pom.xml          ← root reactor, lists the subprojects as <modules>
  project-a/       ← synced from team A's repo
  project-b/       ← synced from team B's repo
  dispatcher/      ← shared web-tier config (single owner)
```

- The Cloud Manager repo **must carry a root `pom.xml`** with each
  subdirectory in `<modules>` — Cloud Manager builds the reactor, so an
  unsynced-but-listed module breaks the build and a synced-but-unlisted one
  silently never deploys.
- The sync job: on push to the mapped branch, check out the source, strip
  git metadata, clone the CM repo, replace the subdirectory, commit, push.
  **Branch mapping is per-repo configurable** (e.g. team `dev` →
  CM `development`).
- Credentials: the samples use two secrets (`MAIN_USER`/`MAIN_PASSWORD`) —
  for the Adobe-hosted repo these are the **Access Repo Info** Git
  credentials from [[cloud-manager-git-repositories]], with the same
  regeneration-invalidates-everywhere caveat (every team's sync job breaks
  at once).

### The deletion trap

Adobe's sample scripts stage with plain **`git add`**, "assuming removals
are handled" — depending on Git config that **does not stage deletions**, so
a file removed from a team repo lives on in the CM subdirectory and keeps
deploying. Use **`git add --all`** in the sync job.

### Onboarding a new team repo — three steps

1. add the sync action/job to the new repo,
2. **run it once** so the code exists in the CM repo,
3. add the new directory to the root `pom.xml` modules.

(Do 2 before 3 — a module entry pointing at a not-yet-synced directory
fails the next pipeline of *everyone*.)

## Enterprise branching and flow

Git Flow-shaped, enforced through PRs with quality-gate validation:

- **feature branches** → merged to a **development branch** when mature →
  merged to the **stable release branch** when validated.
- **Push to stable** → automation triggers the **production pipeline**
  (stage deploy → tests/audits → zero-downtime production deploy).
- **Push to development** → synced to the CM repo's development branch, but
  the **non-production pipeline is started explicitly via the API/CLI**
  (`pipeline:create-execution`) — dev deploys don't auto-run on sync.
- Local work happens against the **AEMaaCS SDK** (local author/publish/
  dispatcher, [[aemaacs-sdk]]) before anything is pushed.

## Why governance is load-bearing, not boilerplate

The production pipeline builds **all teams' code together in one reactor**:
one team's failing quality gate, broken unit test or oversized index change
blocks every team's release. Hence Adobe's insistence on a shared governance
model/standards, and two structural consequences:

- the **dispatcher configuration is a single shared artifact** — the web
  tier can't be split per team, so it lives in the shared root repo with
  one owning team/process gatekeeping changes;
- cross-team conventions (package naming, `-custom-N` index names, shared
  dependency versions in the root pom) are release-blocking concerns, not
  style preferences.

## Exam checklist

- Many customer repos → subdirectories of one CM repo via CI sync; root
  reactor `pom.xml` lists them as modules.
- Onboard: sync job → run once → then add the module.
- `git add` in sync scripts misses deletions → `git add --all`.
- Stable branch auto-triggers production; development branch syncs but
  needs an **API call** for the non-prod pipeline.
- One reactor = one team can block all teams; dispatcher config is shared
  and single-owner.

## References
- [Working with multiple source Git repositories (Adobe docs, AEMaaCS)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/managing-code/working-with-multiple-source-git-repositories)
- [Enterprise team development setup (Adobe docs, AEMaaCS)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/managing-code/enterprise-team-dev-setup)

Docs-based (both pages read Oct 2026), sync scripts not run.
