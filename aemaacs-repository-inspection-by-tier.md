# Inspecting the repository on AEMaaCS: what's available per environment tier

## The short version

`/crx/de` is not the tool on AEM as a Cloud Service. The replacement is the
**Repository Browser** in the Developer Console, and unlike CRXDE it *is*
available on stage and production.

| Tool | Local SDK | RDE | Dev | Stage | Prod |
|---|---|---|---|---|---|
| CRXDE Lite (`/crx/de`) | yes | — | yes (author tier) | **no** | **no** |
| Repository Browser (Developer Console) | **no** | yes | yes | yes | yes |
| Package Manager (`/crx/packagemgr`) | yes | yes | yes | yes | yes |
| Felix web console (`/system/console`) | yes | no | no | no | no |
| Sling JSON (`.N.json` on a path) | yes | yes | yes | yes | yes |

**The citable statement** is in *AEM as a Cloud Service Development
Guidelines*: *"Customers can access CRXDE lite on the author tier's development
environment, but not stage or production."* Same page names the replacement:
*"the Repository Browser can be launched from the AEM as a Cloud Service
Developer Console, providing a read-only view into the repository for all
environments on author, publish, and preview tiers."*

Footnote for anyone who finds the other page: *Using CRXDE Lite* (developer-tools)
says flatly *"It is not available in AEM as a Cloud Service"*, contradicting the
above on Cloud **dev**. Development Guidelines is the more specific statement and
agrees with the Repository Browser page, so cite that one. Both agree on stage
and production: no.

Corollary: the two are mutually exclusive by design. The Repository Browser is
*"**ONLY** available on AEM as a Cloud Service environments"* — it does not
exist on the local SDK, where you use CRXDE. So "browse the repo" means a
different tool depending on which side of the SDK/Cloud line you're on, and a
runbook that says "open CRXDE" is wrong for half your environments.

## Don't confuse this with the 6.5 quickstart case

On AEM 6.5, `/crx/de` loading but reporting itself disabled is a *fixable*
configuration problem — the DavEx backend has no OSGi config; see
[production-ready-mode-crxde.md](production-ready-mode-crxde.md). That fix has
no analogue on AEMaaCS: there is no Felix console on any Cloud tier, so there
is nothing to configure even if the cause were the same. On Cloud it is a
product decision, not a broken component — stop debugging and switch tools.

## Repository Browser: what you actually get

- Read-only view of resources and properties. No editing, no node creation, no
  property changes — if your task needs a write, this is not the path.
- Covers **Production, Stage, and Development**, across **Author, Publish, and
  Preview** services.
- It shows *"resources and properties your user has access to"* — so it honors
  JCR read ACLs rather than bypassing them. Relevant if you are using ACLs to
  scope what someone can see: a deny genuinely hides the node here.
- Access to Publish and Preview tiers is *"limited by default, reducing the
  available resources"* unless the user holds the appropriate administrator
  role.
- Reached through the Developer Console, which is gated by Cloud Manager role
  membership — an AEM author login alone is not enough.

## When the Repository Browser isn't enough

Read-only and per-node, so for anything bulk or offline:

- **Package Manager** — filter the path, build, download the zip, inspect
  locally. The most direct substitute for browsing a subtree, available on
  every tier, and the only one of these that gets content *out*.
- **Sling JSON** — `…/content/site/en.3.json` on the author URL. Good for spot
  checks; don't rely on `.infinity.json`, it's capped on large trees.
- **Developer Console** beyond the browser — bundles, OSGi components and
  configurations, health checks, Java stack traces, query and index
  diagnostics. This is what replaces `/system/console`, which does not exist on
  any Cloud tier.
- **Cloud Manager Content Copy** stage → dev when you really need to click
  around a tree interactively.

## Diagnostic order when someone says "I can't see CRX on stage"

1. Stage — expected, by design. Point them at the Repository Browser.
2. Cloud **dev** — also expected under Adobe's current documentation; don't
   burn time on it, use the Repository Browser there too.
3. Repository Browser missing as well — that's a **Cloud Manager role**
   problem (no Developer Console access), not an AEM permission problem, and
   it is fixed in the Admin Console, not in AEM.
4. Nodes visible but sparse — read ACLs, or the Publish/Preview default
   limitation above. The tool is filtering, not failing.

Adobe docs read September 2026. The Cloud-dev contradiction is between Adobe's
own pages as of that date, not a version difference.
