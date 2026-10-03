# Cloud Manager Git repositories: Adobe-hosted, your own GitHub, or external

Cloud Manager pipelines build from a Git repository, and there are three ways to give them one.
Picking the wrong one costs a migration later, and one of them has a hard platform limit that is
easy to miss.

| Type | Where the code lives | How Cloud Manager gets access |
|---|---|---|
| **Adobe repository** | `git.cloudmanager.adobe.com`, managed by Adobe | You push to it with Git credentials from *Access Repo Info* |
| **Private repository** ("BYO GitHub") | Your own repo on **github.com** | GitHub app + an ownership challenge file |
| **External repository** | Any Git host reachable over HTTPS | Repository URL plus stored credentials |

Adobe repository and private repository are not alternatives you combine: with a private repo,
pipelines read GitHub directly and there is nothing to push to Adobe. Keeping both means two
sources of truth for the same code.

## Private repository (BYO GitHub)

**The limit to check first: public GitHub only.** Adobe states plainly that *"this feature is
exclusive to public GitHub. Support for self-hosted GitHub is not available."* A GitHub Enterprise
Server install cannot use this flow no matter how it is reached — that case needs an *external*
repository instead. Worth confirming before designing around it, because an SSH host alias
(`git@github.mycompany:org/repo.git` in `~/.ssh/config`) makes a plain github.com repo look
self-hosted in `git remote -v`, and a real GHE install look routine.

Setup:

1. A GitHub **organization owner** installs the Adobe app at
   `https://github.com/apps/cloud-manager-for-aem` and grants it access to the repository.
   Not a developer-level action — it needs org ownership.
2. Cloud Manager → **Repositories** → **Add Repository** → **Private Repository**. The repository
   URL must end in `.git`.
3. The *Private Repository Ownership Validation* dialog appears → **Generate** → copy the
   **Secret file content**. **It is displayed once.** Close the dialog without copying and the only
   way forward is regenerating, which invalidates the previous secret.
4. Commit that content to **`.well-known/adobe/cloud-manager-challenge`** on the repository's
   **default branch**.
5. Click **Validate**.

Until validation the repository shows a red icon and cannot be used by any pipeline. After it,
Cloud Manager creates full-stack code-quality pipelines for pull requests and reports results back
as GitHub checks — the actual payoff, since quality gates move to PR time instead of post-merge.

**Ordering trap:** the challenge is fetched from the default branch, so a repository with no
commits (a fresh `main` that was never pushed) cannot be validated. Land the initial commit first.

**One token per path:** the challenge value is issued per Cloud Manager program. A repository
validated against two programs cannot hold both secrets at that single path.

## Adobe repository

Created from **Program Overview → Repositories → Add Repository → Adobe Repository**; the URL is
generated (`https://git.cloudmanager.adobe.com/<org>/<repo>/`).

Credentials come from **Program Overview → Pipelines card → Access Repo Info**, visible to the
**Developer** and **Deployment Manager** roles. Three things that trip people up:

- Cloud Manager Git has its **own username/password pair**. Adobe ID and SSO credentials do not
  authenticate against it.
- The password must be generated, and **generating a new one invalidates the old immediately** —
  including on every machine and CI job still using it.
- Stale cached credentials (macOS keychain, Windows credential manager) are the usual cause of
  sudden auth failures; Adobe has a KB article for exactly this.

To connect an existing repo, add Adobe as a *second* remote rather than replacing `origin`:

```bash
git remote add adobe https://git.cloudmanager.adobe.com/<org>/<repo>/
git push adobe main
```

The *Access Repo Info* dialog prints these commands pre-filled for the program.

## AMS: syncing a customer-managed Git with the Adobe repo

The AMS *Git integration* page (video hub, light on text) frames the
dual-remote workflow as five scenarios: **initial sync** → **branching
aligned to pipelines** (one branch per production/non-production pipeline)
→ **feature branches** validated through the non-production pipeline →
**release prep in the customer repo, synced for stage/prod deployment** →
**release tags synced *back* from Cloud Manager to the customer repo** for
deployment visibility (Cloud Manager generates release versions/branches
itself, so the tags only exist Adobe-side until you pull them). Explicitly
conventions-not-mandates; no tooling prescribed — the AEMaaCS-era CI-sync
automation for the same problem is
[cloud-manager-multi-repo-team-setup.md](cloud-manager-multi-repo-team-setup.md).

### Jenkins recipe for the 1:1 mirror (community)

For the simple case — one internal repo mirrored to the Adobe repo, no
subdirectory aggregation — a community-documented **zero-script Jenkins
freestyle job** does it with the Git plugin alone:

- **SCM → Git with TWO repositories** in one job: source (on-prem) and
  destination (Cloud Manager), each with a distinct **Name** (repo ID —
  only the first checkout defaults to `origin`); *Branch Specifier* = the
  on-prem branch to watch.
- **No Build Environment config, no Build steps.** The whole job is SCM +
  post-build.
- **Post-build Actions → Git Publisher** pushes the checked-out state to
  the Cloud Manager branch; its **Target remote name must equal the Name**
  given to the CM repository in the SCM section (the easy misconfiguration).
  Choice of merge vs force push to the CM branch.
- Automation is optional Build Triggers: a webhook on the on-prem repo
  (push events) or polling; without it the job is a manual sync button.

Mirror-push vs the aggregation pattern: this replays the *whole repo* onto
the Adobe remote (fits the 1-repo AMS norm), while multi-repo setups need
the copy-into-subdirectory scripts — and that pattern's `git add --all`
deletion trap doesn't exist here, since Git Publisher pushes refs, not a
re-staged working tree. Source: Chintalapudi & Chugh, Medium, Jul 2020 —
UI labels may have drifted since.

## Managing the repository list (lifecycle facts)

From the AEMaaCS *Managing Repositories* page:

- **Adding needs Deployment Manager or Business Owner.** Hard ceiling: **300 repositories across
  all programs in the IMS organization**.
- Validation method differs by host and is the quick way to tell the flows apart:
  **github.com = Adobe GitHub app + secret challenge file, no webhook**;
  **GitHub Enterprise Server, GitLab, Bitbucket, Azure DevOps (external) = personal access token +
  webhook**.
- UI actions per repo: Copy URL, View & Update (name/description), Delete, Access Repo Info —
  plus **Check Branches / Create Project on Adobe repos only**.
- **Deletion**: blocked while any pipeline references the repo; after deletion the repo cannot be
  linked again and **its name is burned forever** — re-creating one with the same name fails with
  *"Repository name should be unique within organization"*, and there is no restore. Name
  accordingly (no throwaway names you'll want back).
- **Git submodules are supported** for pulling secondary branches/repos into one build — the
  lighter-weight alternative to the CI-sync aggregation in
  [cloud-manager-multi-repo-team-setup.md](cloud-manager-multi-repo-team-setup.md).

## Sources

Read from Adobe documentation, September 2026 — steps not yet performed end to end on a live
program, so treat the UI labels as current-as-documented rather than screen-verified.

- [Managing Repositories in Cloud Manager (AEMaaCS)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/managing-code/managing-repositories)
- [Git Integration with Cloud Manager (AMS; video tutorials for the five sync scenarios)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-manager/content/managing-code/git-integration)
- [Integrating internal Git with Cloud Manager using Jenkins (community blog)](https://experienceleaguecommunities.adobe.com/adobe-experience-manager-sites-8/integrating-internal-git-with-cloud-manager-using-jenkins-aem-community-blog-seeding-221497) → [full article on Medium](https://chintalapudi4.medium.com/integrating-internal-git-with-cloud-manager-using-jenkins-c2dd2d2727f9)
- [Add Private Repositories in Cloud Manager](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-manager/content/managing-code/private-repositories)
- [Github.com repositories — tutorial with video](https://experienceleague.adobe.com/en/docs/experience-manager-learn/cloud-service/cloud-manager/byogithub) ([video](https://video.tv.adobe.com/v/3429302/?learn=on))
- [AEM GEMs: Integrating Private GitHub Repositories in AEM Cloud Manager, 31 July 2024](https://experienceleague.adobe.com/en/docs/events/experience-manager-gems-recordings/gems2024/private-github-for-aem-cloud-manager) ([video](https://video.tv.adobe.com/v/3432350?learn=on))
- [Add an Adobe Repository](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-manager/content/managing-code/adobe-repositories)
- [Add External Repositories](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-manager/content/managing-code/external-repositories)
- [Repository Access Information](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/managing-code/accessing-repos)
- [Pull Request Checks for Private Repositories](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/managing-code/github-check-config)
- [KB: Cloud Manager Git authentication failures from expired or cached credentials](https://experienceleague.adobe.com/en/docs/experience-cloud-kcs/kbarticles/ka-35958)