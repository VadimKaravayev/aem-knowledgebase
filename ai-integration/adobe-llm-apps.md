# Adobe LLM Apps — brand actions + widgets inside ChatGPT/Claude, served as an MCP server

**Adobe LLM Apps** (Adobe CX Enterprise, **Beta**, `experience.adobe.com/llm-apps`, Beta access via llm-apps-beta@adobe.com) lets a brand expose **actions** ("Schedule a test drive", "Browse products") and **visual widgets** (product cards, booking forms, store locators) inside supported AI assistants — Adobe names **ChatGPT and Claude**. Deployment produces an **MCP server URL**; the assistant connects to it, discovers the actions by description and invokes them. Written from the Adobe doc set (overview, architecture, create-app, handler, widgets, deploy pages); overview last updated 2026-10-23. Beta: UI and workflow may change; nothing here is production-verified.

## The five concepts

| Concept | What it is | Where it lives |
|---|---|---|
| **LLM App** | the branded assistant: groups all actions, deploys as one unit | LLM Apps UI |
| **Action** | one capability; the LLM picks it when the request matches its **description** | metadata in LLM Apps UI, code in the **handler repo** |
| **Action handler** | server-side function: validates input, calls your APIs, returns text + structured data | GitHub handler repo → runs on **Adobe I/O Runtime** ([[app-builder]]) |
| **Widget** | the visual response (card, carousel, table) — an **EDS block** | GitHub widget repo = an **Edge Delivery Services** project ([[edge-delivery-services-overview]]) |
| **MCP server** | the endpoint exposed per environment after deploy | shown under *Test the app* → *Copy URL* |

An app has one handler repo + one widget repo; adding an action adds folders, never repos. Text-only actions work everywhere; widget rendering depends on the LLM platform.

## Request flow

user prompt → assistant matches an action by description → MCP call → handler fetches from your systems → returns `content` (text for the model) + `structuredContent` (data for the widget) → widget renders → conversation continues. **If the handler's output shape changes, the widget breaks**; change handler, widget and tests together.

## Setup (create-app wizard)

- Prerequisite in all cases: **an AEM program** (Cloud Manager program, [[cloud-manager-program-types]]) and an **analytics data region (US/EU)** — **both immutable after creation**.
- **Adobe managed infrastructure** ("Let us handle it"): Adobe creates and hosts both repos; 1–2 min setup; code access on request. Two wizard steps.
- **Own repositories** ("I'll set it up myself"): two **empty** GitHub repos (**no README/license/.gitignore** — validation writes the boilerplate), the **Adobe LLM Apps GitHub App** installed with *Only select repositories*, and for the widget repo **AEM Code Sync** installed + yourself as EDS site administrator, validated inside the wizard. GitHub is the only code source; "Use Cloud Manager Edge Delivery Services" is *Coming soon*.
- Creating the app does **not** generate actions; start that from the app page (analyze the website, or create manually). Generated actions use **sample data** — wire the handlers to real systems before customers see it.

## Handler contract (handler repo)

```
actions/<action-slug>/index.js      # slug = code identifier with hyphens: search_products → search-products
actions/<action-slug>/data.json     # sample data
test/actions/<action-slug>.test.js  # Jest by default
```

```js
const { getVariable } = require('@adobe/llm-apps-runtime');
module.exports = async ({ query = '' } = {}, extra) => ({
  content: [{ type: 'text', text: 'Found 3 products.' }],          // always required; what the model reads
  structuredContent: { arguments: { query }, records: [ /* … */ ] }, // required when a widget exists; plain object, never a bare array
  // isError: true on failure
});
```

- **App variables** come via `getVariable(extra, 'NAME')`, **not `process.env`**; set per environment (Stage/Prod independent), applied on the **next deployment**. The UI can only *add* variables and Adobe says **do not store credentials there** — keep secrets in server-side handler code / your own secret store; never in widget JS.
- Runtime validation covers required fields, primitive types, enums, array items — **not full JSON Schema**; check lengths, ranges, nested objects yourself.
- `structuredContent` reaches the host **and the model may see it** → return only authorized, needed data; no stack traces, internal hosts or upstream bodies.
- Three states to implement: results / no results (`records: []`) / failure (`isError: true` + safe text). Don't touch runtime files such as `entry.js`.
- An action with UI metadata but **no handler file still deploys** — a stub answers until you add code.

## Widget contract (EDS repo)

```
blocks/<action-slug>/<action-slug>.js + .css   # one EDS block per action
scripts/aem-embed.js, scripts/llmapps-sdk.js   # SDK entry points, do not import the SDK inside a block
/eds-widgets/<action-slug>-demo                # widget page; the Widget URL in action metadata is the source of truth
```

```js
export default async function decorate(block, bridge) {
  const live = bridge?.isConnected && bridge.hostContext?.preview !== true;
  const result = live ? await bridge.toolResult : null;      // resolves ONCE; later results via bridge.onToolResult(cb)
  const records = live ? result?.structuredContent?.records ?? [] : PREVIEW_RECORDS;
  // render; show an error state when result?.isError
}
```

Bridge API (`@adobe/llmapps-sdk`): `isConnected`, `hostContext` (`preview`, `theme`), `toolResult`, `onToolResult`, `applyHostStyles()`, `onContextChange`, `sendMessage(text)`, `callTool(codeIdentifier, args)`, `autoResize(block)`. Rules: **guard every call with `isConnected`** (a bridge can exist unconnected — awaiting `toolResult` then hangs forever); `callTool` takes the **code identifier** (`search_products`), needs the target action's *app visibility* on, and does **not** re-render your widget; treat handler output as untrusted (`textContent`, validate URLs before `href`/`src`). Local preview: `npm i -g @adobe/aem-cli && aem up` → open the full widget path on `:3000`; direct EDS preview never exercises the host handshake.

## Deploy and environments

- Deploy from the App Detail page → pick **Stage** or **Production** → four steps: Preparing → Start deployment → Build app (npm install + build) → **Publish to Adobe I/O Runtime**. History keeps the last 10.
- **Per-environment, nothing cascades**: Stage ≠ Production; variables are set per environment.
- **Saving is not deploying**: action metadata, visibility switches, authentication, app variables and analytics settings take effect only on the **next deployment** to that environment.
- **Widgets publish through EDS, not through app deployment.** If Stage and Prod share EDS URLs, a widget push changes both — there is no isolated Stage copy of the EDS site.
- Testing: copy the environment's MCP URL → create the plugin/connector in ChatGPT or Claude (separate Adobe guides per platform).
- Analytics: built-in, powered by **Customer Journey Analytics** — trigger counts, success/abandonment rates, top prompts, visibility scores; needs the analytics runtime/SDK level (see Adobe's "Upgrade an existing app").

## Design / exam traps

- "Where do secrets go?" — not app variables (UI add-only, Adobe forbids credentials), not widget JS; server-side handler code.
- "Changed the action description/visibility, why does Claude still see the old one?" — not redeployed.
- "Stage widget test broke production" — shared EDS URLs; EDS publication is independent of app environments.
- "`await bridge.toolResult` never returns" — unconnected bridge (direct EDS preview or standalone); guard with `isConnected`.
- "Second search result doesn't update the widget" — `toolResult` resolves once; subscribe with `onToolResult`.
- "Can I pick a different AEM program later?" — no; program and analytics region are permanent.
- Handler returns an array as `structuredContent` — rejected; must be a plain object.

## References

- Overview: https://experienceleague.adobe.com/en/docs/llm-apps/using/overview
- Architecture: https://experienceleague.adobe.com/en/docs/llm-apps/using/build-first-app/app-architecture
- Create app: https://experienceleague.adobe.com/en/docs/llm-apps/using/build-first-app/create-app
- Customize handler: https://experienceleague.adobe.com/en/docs/llm-apps/using/customize-generated-app/customize-handler
- Widgets: https://experienceleague.adobe.com/en/docs/llm-apps/using/customize-generated-app/widgets
- Deploy: https://experienceleague.adobe.com/en/docs/llm-apps/using/deploy-and-test/deploy-your-app
- SDK: https://www.npmjs.com/package/@adobe/llmapps-sdk
