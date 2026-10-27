# Experience Modernization Agent (EDS migration console)

Adobe's agentic migration tool for **moving an existing site onto Edge
Delivery Services**: a hosted chat console at **aemcoder.adobe.io** that
imports pages and styles from a live URL, writes block code into your EDS
GitHub repo, and pushes the content to **Document Authoring (da.live)** or
**AEM Author (Universal Editor / Crosswalk)**. Part of the "Brand Experience
Agent" family under *AI in AEM*. Condensed from the Experience League
getting-started (updated 2026-09-15), AEM-authoring variant (2026-06-17),
console (2026-06-17), overview and prompting guide (both 2026-06-05).

Companion notes: [[edge-delivery-services-overview]] (what EDS is, the two
authoring models), [[aemaacs-architecture-overview]] (EDS Publish as a second
origin), [[cloud-manager-git-repositories]] (external repos and access
tokens, which this tool depends on), [[edge-functions]] (where custom
server-side logic goes after migration).

---

## 1. What it is, in one paragraph

A web-based AI development environment with **no local setup**: you connect
an EDS GitHub project, type "Migrate the page https://example.com/about",
and the agent scrapes the page, separates content from presentation, maps
sections to blocks, generates block JS/CSS, and shows a live preview. Content
is then **uploaded** (to da.live or AEM) and code is **pushed as a PR**.
Developers keep control: *every* console change goes through GitHub review
before it can deploy. The console is **generally available** as self-service,
but **access is requested through your account manager**; the page lists no
licence or price. Adobe's "AOE delivery" (engineering-led migration) is a
separate, optional engagement for complex sites.

## 2. Two target modes, two repo templates

| | Document Authoring target | AEM authoring (Universal Editor) target |
|---|---|---|
| Repo template | `adobe/aem-boilerplate` | `aem-block-collection-xwalk` (**not** the standard boilerplate) |
| Content lands in | `https://da.live/#/{org}/{repo}` | AEM Author, as a content package uploaded to the site created from the **"AEM Site with Edge Delivery Services Template"** (from `aem-boilerplate-xwalk` releases) |
| Upload action | Content view → **Upload content** → select files → **Upload** | Content view → **Upload content** → package from selected pages → **Upload to AEM** (site/host pre-filled from project settings, images optional) |
| Extra prep | none beyond the GitHub apps | finish the Universal Editor tutorial up to "create a site", confirm `fstab.yaml` points at AEM host + git owner/repo, **delete `paths.json`** and commit to `main` *before* installing the apps; keep the generated site name (it is the unique identifier) |
| Workspace views | Preview, Document, HTML | plus Markdown and **JCR XML** views and an **Errors** action |

Trap: the AEM-authoring flow uses the *xwalk block collection* as the template.
Starting from the plain boilerplate and "adding AEM later" is not the documented
path.

## 3. Prerequisites (both modes)

- An EDS repo (existing or from the template) with **two GitHub apps**:
  **AEM Code Sync** (EDS syncs code; already there if you did the developer
  tutorial) and **AEM Code Connector** (lets the console inspect/write code).
- A **Cloud Manager access token with write access** to repository code plus
  the permissions from step 8 of "Add external repositories". Existing tokens
  without write access are edited under **Repositories → Manage Access
  Tokens**; repos revalidate automatically.
- Non-GitHub hosts (GitHub Enterprise self-hosted, Adobe-hosted repos, GitLab,
  Bitbucket, Azure DevOps) must be **onboarded in Cloud Manager first**
  (Bring Your Own Git).
- Adobe ID for sign-in; console access granted on request.
- Source site behind a firewall: **allowlist** 34.228.136.112, 54.90.51.39,
 3.224.194.242 (from the Troubleshooting section; recheck before
  quoting to a customer). Authenticated/intranet content cannot be scraped at
  all; Adobe points to SLICC for that.

## 4. The procedure (DA variant, 8 steps)

1. Prepare repo (template + both apps).
2. Open aemcoder.adobe.io → sign in. It starts in **demo mode** with a bottom
   banner.
3. **Switch site** in the banner → **Authorize AEM Code Connector** (GitHub
   creds) → enter the site's **Preview URL** (`https://<branch>--<repo>--<org>.aem.page`,
   or search authorized projects) → **Checkout to workspace** → **Replace
   workspace**. Leaving demo mode without connecting forces a connection next
   visit.
4. Prompt: `Migrate the page https://…` — takes **several minutes**; select the
   page in **Workspace** to preview.
5. Prompt: `Import the general styles from https://…` → **Refresh preview**.
6. **Content** view → **Upload content** → confirm org/repo (manual if the repo
   has no `fstab.yaml`) → **Upload** → **View in AEM** / da.live.
7. **Changes** view (branch icon) → `+` to stage untracked files → **Push** →
   new PR (default) or current branch → **Confirm** → **View PR**.
8. In da.live open the page → paper-plane icon → **Preview**. If code went to a
   branch other than `main`, **edit the preview URL to that branch** or you
   will see unstyled content and think the import failed.

## 5. Console anatomy (what the exam-style "which panel" questions mean)

- **Chat panel**: history + prompt; **Clear chat** (drops context), **Download
  chat** (markdown export). Prompt input has **Plan mode** (outline only, no
  file changes) vs **Execute mode**; attachments (designs, screenshots, specs);
  prompt queueing.
- **Workspace panel**, view picker:
  - *Content views*: file browser; Preview / Document / HTML (+ Markdown, JCR
    XML for AEM); Refresh, responsive modes, **Inspect mode** (click an element
    to add it to the prompt), open in window, Delete (workspace only, never
    published content), Errors (AEM), Upload content.
  - *Code views*: **Files** and **Changes** (diffs); per-file **Add to chat**
    and Download.
- **Settings**: connect/reconnect GitHub, switch repo (**uncommitted work is
  lost**), **Library URL** (a `library.json` describing blocks/variants/example
  content), **Site base URL**, agent permissions (fetch site config from
  `admin.hlx.page` with your IMS creds, or a custom IMS token), read-only
  **Figma PAT**, Support, Danger zone (**reset workspace, irreversible**).
- **Git**: Switch Branch, Sync (pull origin), Add/discard/Refresh changes,
  Push (needs staged changes) to new PR or current branch.

## 6. Capabilities and the explicit "no" list

Can: page migration, bulk import, design/token extraction, navigation setup,
scraping, **site catalog** ("Scope site https://…" → templates, block
variants, screenshots), block development with tests (Content-Driven
Development), Figma → block or page (one `node-id` at a time), documentation
search, post-project documentation generation, migration from other CMSs,
legacy AEM, or Figma design systems.

Cannot / manual: authenticated or intranet sources; complex interactive
dynamic content; **HTL-based AEM Publish targets or headless/SPA delivery**
(it only produces EDS); pixel-perfect fidelity; third-party integrations,
commerce, search, MarTech data layers, experimentation, Content/Experience
Fragment isolation, **MSM inheritance**, custom business logic. Those stay
human work (and typically land in [[edge-functions]] or external services).

## 7. Prompting rules Adobe publishes (the ones that bite)

- **One representative page per template first, then bulk import.** Bulk
  import before a single migration fails because the import infrastructure
  does not exist yet. Never mix templates in one bulk run.
- Site-wide design before per-block styling; prefer **block variants over
  config cells**; **max four cells per block-table row**.
- Keep project rules in `INSTRUCTIONS.md` / `CONTEXT.md` and tell the agent to
  read them at the start of each chat; do not expect a long chat to retain
  early instructions, start fresh with a warm-up prompt instead.
- Figma: one block/frame by `node-id`, never the whole file; dynamic features
  (search, calculators, store locators) will not come out of a Figma
  migration.
- Run the agent's browser validation / tests before opening a PR; skip CDD
  only for trivial CSS tweaks.

## 8. Quick answers

- "Can we migrate our AEM 6.5 site to EDS automatically?" → the agent gets you
  blocks, styles and content from the *rendered public site*; everything in the
  "cannot" list is still a project. Treat it as a scaffold accelerator, not a
  migration.
- "Do we need a local setup?" → no, the console is hosted; you do need the two
  GitHub apps and a Cloud Manager token with write access.
- "Where does content end up?" → da.live by default; AEM Author only if you
  started from the xwalk template and created the site from the EDS site
  template.
- "Why is the preview unstyled?" → code was pushed to a non-`main` branch;
  point the preview URL at that branch.
- "Who approves what the AI wrote?" → nobody in the console; it is a PR like
  any other and your repo's review rules apply.

## References
- [Getting started (DA)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/ai-in-aem/agents/brand-experience/modernization/getting-started)
- [Getting started (AEM authoring / Universal Editor)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/ai-in-aem/agents/brand-experience/modernization/getting-started-aem-authoring)
- [Experience Modernization Console](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/ai-in-aem/agents/brand-experience/modernization/console)
- [Prompting guide](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/ai-in-aem/agents/brand-experience/modernization/prompting-guide)
- [Overview](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/ai-in-aem/agents/brand-experience/modernization/overview)
- GitHub apps: [AEM Code Connector](https://github.com/apps/aem-code-connector), [AEM Code Sync](https://github.com/apps/aem-code-sync)
- [Bring Your Own Git](https://www.aem.live/developer/byo-git), [Document Authoring](https://da.live)
