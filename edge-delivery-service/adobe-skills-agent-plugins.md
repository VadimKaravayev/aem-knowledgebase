# Adobe Skills: agent plugins for AEM, EDS and other Adobe products

[adobe/skills](https://github.com/adobe/skills) (Apache-2.0) is Adobe's
official repository of **skills for AI coding agents**, packaged as a Claude
Code plugin marketplace named **`adobe-skills`**. The same manifests are read by
GitHub Copilot CLI, and the skills install into other agents via
`npx skills` or `gh upskill`. Checked against the repo's
`.claude-plugin/marketplace.json` and the EDS plugin's file tree with Claude
Code 2.1.x.

Companion notes: [[edge-delivery-services-overview]],
[[edge-delivery-media-bus]].

---

## 1. Which plugin for which project

| Plugin (`<name>@adobe-skills`) | Use it for |
|---|---|
| `aem-edge-delivery-services` | **EDS code repos**: content-driven development, blocks, testing, content modeling, code review, page import/migration |
| `aem-eds-content-ops` | EDS content work: page audits, SEO, GEO (AI search), WCAG, bulk metadata, structured data, sitemap validation, content diffing |
| `aem-project-management` | EDS project lifecycle |
| `aem-cloud-service` | **AEMaaCS Java stack**: components, Dispatcher authoring/hardening, workflows, code assessment and migration from legacy AEM, content distribution, RDE |
| `aem-6-5-lts` | **AEM 6.5 LTS / AMS**: Dispatcher, workflows, replication agents and the Replication API, activation, troubleshooting, project bootstrap |
| `app-builder` | Adobe App Builder (incl. Workfront UI extensions); the only plugin with a Cursor manifest so far |
| `commerce-app-management`, `commerce-app-migration`, `commerce-app-review` | Adobe Commerce App Builder apps: scaffold, migrate from starter kits, pre-Exchange review |
| `adobe-analytics`, `adobe-cja` | Analytics / CJA practitioner skills; need the product's MCP server and IMS auth headers |
| `workfront` | Workfront Planning architecture, Fusion scenario optimizer |
| `run-workflow` | Firefly workflows via MCP; needs an enterprise Firefly entitlement |
| `web` | Browser automation and page analysis with `playwright-cli` (CDP, bot-protection probing, DOM capture) |
| `stardust` | Guided website redesign; depends on the external `impeccable` plugin |

Pick by stack: an EDS repo wants `aem-edge-delivery-services` (plus
`aem-eds-content-ops` for content work), a Maven/Java AEMaaCS repo wants
`aem-cloud-service`, an on-prem/AMS repo wants `aem-6-5-lts`. Installing the
wrong AEM flavour adds skills whose advice does not apply (Dispatcher and
OSGi guidance in an EDS repo, for example).

## 2. The EDS plugin

Skills only: **24 skills, no hooks, no MCP servers, no slash commands** (so
installing it runs no code on its own). Groups:

- **Build**: `content-driven-development`, `analyze-and-plan`,
  `building-blocks`, `content-modeling`, `testing-blocks`, `code-review`,
  `block-collection-and-party`, `block-inventory`
- **Tools and docs**: `aem-cli`, `docs-search` (searches aem.live),
  `find-test-content`
- **Import/migration**: `page-import`, `scrape-webpage`, `page-decomposition`,
  `identify-page-structure`, `generate-import-html`, `preview-import`,
  `snowflake`
- **Authoring**: `da-content`, `da-auth` (Document Authoring), `ue-component-model`
  (Universal Editor), `figma-to-content`, `authoring-analysis`
- **Setup/handoff**: `create-site`, `slicc-handoff`

The repo also ships `evals/` per plugin (task + criteria) usable with
`claude plugin eval`.

## 3. Installing

Inside Claude Code:

```
/plugin marketplace add adobe/skills
/plugin install aem-edge-delivery-services@adobe-skills
```

From a shell, **per project** (so everyone who opens the repo is offered the
same skills):

```sh
claude plugin marketplace add adobe/skills --scope project
claude plugin install aem-edge-delivery-services@adobe-skills --scope project
```

This writes `.claude/settings.json` in the repo:

```json
{
  "extraKnownMarketplaces": {
    "adobe-skills": { "source": { "source": "github", "repo": "adobe/skills" } }
  },
  "enabledPlugins": { "aem-edge-delivery-services@adobe-skills": true }
}
```

Skills load in the next session (restart, or `/reload-plugins`).

Other agents: `copilot plugin marketplace add adobe/skills` then
`copilot plugin install <name>@adobe-skills`; `npx skills add adobe/skills --all`;
`gh extension install ai-ecoverse/gh-upskill && gh upskill adobe/skills --all`.

Codex: `codex plugin marketplace add adobe/skills`, then `/plugins` in the
Codex CLI → search `aem-edge-delivery-services` → install. `gh upskill` can
take only the EDS skills:
`gh upskill adobe/skills --skills-path plugins/aem/edge-delivery-services --all`.

Skills apply automatically; name a skill in the prompt when you want a
specific one. aem.live highlights six as the entry points: `create-site`,
`content-driven-development` (the default for building or changing blocks),
`page-import`, `docs-search`, `block-inventory`, `block-collection-and-party`.

## 4. Setting up an agent for an EDS project (aem.live guidance)

Condensed from aem.live's
[Developing with AI Tools](https://www.aem.live/developer/ai-coding-agents).

- **Rules file.** The current `aem-boilerplate` ships an **`AGENTS.md`**
  (structure, setup commands, conventions, block patterns); older forks don't
  have one, so copy it from the boilerplate. Claude Code reads **`CLAUDE.md`**,
  so keep both (a `CLAUDE.md` that imports `@AGENTS.md` works).
- **`llms.txt`.** `https://www.aem.live/llms.txt` lists the key docs in an
  AI-friendly format. Agents don't fetch it on their own; point to it in the
  rules file. Its links use **`.md` URLs** (every aem.live page is served as
  markdown, e.g. `/developer/tutorial.md`), which are far cheaper to fetch
  than the HTML.
- **Searching.** Asked about "AEM", agents mix in the Java/JCR/OSGi stack;
  "Edge Delivery" brings up Cloudflare/Akamai/Fastly edge compute, and "EDS"
  brings up Ehlers-Danlos syndrome. Say **"search www.aem.live"**, or index
  `https://www.aem.live/docs/` where the tool supports custom docs.
- **Letting the agent see the page.** Screenshots of `localhost:3000`, or a
  browser tool the agent drives (`playwright-cli`, `agent-browser`, Chrome
  DevTools/Browser MCP). Without one, the agent guesses what the page looks
  like.
- **AI pull request reviews** fit the EDS flow (everything goes through PRs):
  GitHub Copilot code review (no setup), Claude Code via GitHub Actions,
  Codex code review via its GitHub app.
- **CLI tools agents use well**: `rg`, `jq`, `gh`, `curl`, `ast-grep` (`sg`),
  `httpie`, `fzf`, `fd`, `bat`.
- **MCP servers** (each costs context; disable unused ones): Context7 (has
  aem.live's `llms.txt` indexed), the unofficial **helix-mcp**
  (`cloudadoption/helix-mcp`: docs search, block starters, admin tools), the
  unofficial **DA MCP** (Document Authoring content, early access), Browser MCP.
- **Hosted option:** Adobe's **AEM Experience Modernization Agent** combines
  the site-creation, migration and block-development skills with a hosted
  console, for migrations.
- **When the agent keeps making the same mistake**, ask it to update the
  skill (or the rules file) so the fix sticks.

## 5. Gotchas

- **EDS repos serve every committed file.** `.claude/settings.json` is safe in
  the boilerplate only because its `.hlxignore` excludes dotfiles (`.*`); check
  that line exists before committing agent config to an EDS repo.
- **Copilot CLI uses bare skill names** (no plugin prefix), so two installed
  plugins with a same-named skill (e.g. a `code-review` skill) are ambiguous
  there, and Copilot does not resolve plugin `dependencies`: install them
  explicitly (`stardust` needs `impeccable`).
- **The marketplace is a moving `main` branch.** Skill content changes on
  refresh; plugin versions in the manifest are not always bumped (the EDS
  plugin stays at `1.0.0` while its skills carry their own changelogs).

## 6. What this means for AEM

These are agent instructions, not AEM software: nothing is installed into an
AEM instance or an EDS site. They are most useful where AEM knowledge is
scattered (EDS lives on aem.live, not Experience League) and the agent would
otherwise guess. Treat their advice like any other doc and check it against
the version you run; this KB stays the place for verified, counter-intuitive
findings.
