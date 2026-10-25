# Adobe I/O CLI (`aio`) — one-page overview and plugin map

`aio` is Adobe's single command-line tool for its developer platform: a Node.js program built on **oclif**, installed with `npm install -g @adobe/aio-cli`, and extended by **plugins that each own a command namespace** (`aio cloudmanager …`, `aio aem …`, `aio app …`). For AEM work it is the scripting surface for Cloud Manager, RDEs, Edge Functions and App Builder. This page is the orientation layer; the plugin-level detail lives in the linked docs.

## Auth

- **Interactive**: `aio auth:login` opens a browser IMS login and stores the token locally. Fine for a developer laptop, useless in CI.
- **Headless / CI**: an **OAuth Server-to-Server** credential from Developer Console, stored as an IMS context in the aio config (for the Cloud Manager plugin the context is `ims.contexts.aio-cli-plugin-cloudmanager`). Credential types: [[adobe-ims-authentication-types]].
- The **JWT credential type is dead IMS-wide**; the Cloud Manager plugin auto-migrates `jwt-auth` contexts, but new setups must be OAuth S2S.
- **Permissions belong to the credential's product profile, not to the person running the CLI.** A credential created by a Developer Console System Admin still needs a Cloud Manager role profile (Developer / Deployment Manager / Business Owner) assigned in Admin Console to do anything beyond read — see [[admin-console-product-profiles]] and the permission matrix in [[cloud-manager-api-cli-sdks]].

## Plugins that matter for AEM

| Plugin / namespace | What it drives | Detail |
|---|---|---|
| `aio cloudmanager` (`@adobe/aio-cli-plugin-cloudmanager`) | Programs, environments, pipelines: list, `create-execution`, `current-execution:advance`/`cancel`, build + runtime variables, `tail-log`/`download-logs`, IP allow lists, content flows | [[aio-cloudmanager-cli]] |
| `aio aem rde` | Sync code and content to a **Rapid Development Environment without a pipeline**: install bundles, content packages, OSGi configs, dispatcher config, `reset` | [[cloud-manager-environments]] |
| `aio aem edge-functions` | `build`, `serve` (local on `127.0.0.1:7676`), `deploy <name>`, log tailing for AEM Edge Functions on the Adobe CDN | [[edge-functions]] |
| `aio app` + `aio console` | Scaffold, run locally, test and deploy **App Builder** apps on I/O Runtime; `console` selects org / project / workspace | [[app-builder]] |
| `aio runtime` | Raw I/O Runtime (OpenWhisk) actions, packages, triggers | [[app-builder]] |
| `aio api-mesh` | API Mesh service (GraphQL federation layer) | — |
| `aio config`, `aio plugins` | Global settings; install/update plugins (`aio plugins:install @adobe/aio-cli-plugin-cloudmanager`) | — |

The Cloud Manager plugin is also published standalone as `adobe-cloudmanager-cli` (npm -g) for CI images that do not want the whole `aio` stack.

## Config model

- `aio config:set` stores global defaults so commands run without repeating identifiers: `cloudmanager_orgid`, `cloudmanager_programid`, `cloudmanager_environmentid`.
- `-j` (JSON output) and `-y` (skip confirmations) are accepted everywhere — the basis of every scripting recipe.
- Cloud Manager plugin: **`--permissions` on any command prints the product profiles allowed to run it** — the fastest answer to "which role do I need for X".
- Exit codes on the Cloud Manager plugin: 1 generic, 2 config, 3 validation, **10 IMS auth, 30 Cloud Manager API** — split CI alerting on 10 vs 30 so an expired credential and a failed pipeline step page different people.

## Where `aio` does *not* reach

- Deploying AEM code to dev/stage/prod still goes through a **Cloud Manager pipeline**; `aio cloudmanager` starts and approves it but does not replace it. Only RDEs take direct `aio aem rde install` pushes.
- App Builder and AEM are **two deployment systems and two release trains** (`aio app deploy` vs Cloud Manager); a change spanning both is not atomic — see [[app-builder]].
- Content Transfer Tool, Admin Console user management and Dispatcher config validation have their own tooling; `aio` is not involved.

## Quick traps

- JWT credentials no longer work; migrate to OAuth S2S before the next token refresh fails in CI.
- A "403" from `aio cloudmanager` is almost always a missing product profile on the *credential*, not on the developer.
- `cloudmanager_programid` / `_environmentid` defaults silently target the wrong environment when copied between projects — print them with `aio config:get` in CI logs.
- RDE commands need the `aio-cli-plugin-aem-rde` plugin installed and the RDE selected via config; a plain `aio` install has no `aem` namespace.
