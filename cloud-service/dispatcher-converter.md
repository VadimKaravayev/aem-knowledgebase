# Dispatcher Converter — 6.x/AMS Apache + Dispatcher config → AEMaaCS structure

One of the code refactoring tools in the cloud transition toolkit ([[cloud-transition-tools-overview]]). It rewrites an existing Apache httpd + Dispatcher configuration from **AEM 6.x on-premise or AMS** into the fixed folder structure and directive subset that **AEM as a Cloud Service** accepts. It is **structural, not semantic**: it makes the config *valid* for the Cloud validator, not *equivalent* behind the Adobe CDN.

## How to run it

- **Adobe I/O CLI**: `aio aem-migration:dispatcher-converter` from `aio-cli-plugin-aem-cloud-service-migration` (the same plugin hosts the Repository Modernizer and Index Converter) — see [[adobe-io-cli-overview]].
- **Standalone**: `@adobe/aem-cs-source-migration-dispatcher-converter` npm package. Same code, same GitHub repo under `adobe`.

## Two input modes

| Mode | Input | Effort afterwards |
|---|---|---|
| **AMS** | the AMS layout: `conf`, `conf.d`, `conf.dispatcher.d`, `conf.modules.d` | low — the well-trodden path, almost no extra config |
| **On-premise** | paths to `httpd.conf`, vhost files and `dispatcher.any` files declared in a `config.yaml` | high — arbitrary layouts convert, but far more lands in the manual list |

## What it changes

- Collapses everything into **a single publish farm**; the author farm and any extra farms are dropped (Cloud dispatcher fronts publish only).
- Replaces hardcoded hostnames/IPs in `/renders` with the Cloud **variables** (`$(PUBLISH_IP)` etc. via the `conf.d/variables` folder).
- Converts Apache `Include` directives to Dispatcher **`$include`** and points `/clientheaders`, `/filters` and `/renders` at the Cloud **default include files**.
- Strips directives the Cloud runtime forbids or ignores: `/statistics`, custom `/allowedClients`, non-standard Apache modules, non-publish listeners.
- Emits the mandatory Cloud layout under `src/`: `conf.d/available_vhosts` + `enabled_vhosts` (symlink pairs), `conf.dispatcher.d/available_farms` + `enabled_farms`, plus `cache/`, `clientheaders/`, `filters/`, `renders/`, `virtualhosts/`, `variables/`, `dispatcher_vhost.conf`.

## The summary report is the work list

Every run writes a markdown **result file** listing each transformation applied and **every item it could not convert**. Treat it as the backlog. Typical leftovers:

- Rewrite rule *semantics* (rules are copied, their behaviour behind the CDN is not checked).
- Custom Apache modules.
- Multiple domains that previously mapped to separate farms — now one farm, so vhost-level differences must be re-expressed.
- Anything environment-specific that lived in the config rather than in variables.

## After conversion

1. Run the **SDK Dispatcher Tools validator** over `src/` ([[aemaacs-sdk]]). It is the same validator Cloud Manager's Build step runs, so a clean local pass means the pipeline will not fail on dispatcher config ([[cloud-manager-pipeline-steps]]).
2. Start the SDK Dispatcher **Docker image** locally against the Quickstart jar and smoke-test caching, filters and rewrites ([[dispatcher-fundamentals]], [[dispatcher-caching-and-invalidation]]).
3. Decide on **validation mode**: the converter targets *legacy* mode. *Flexible* mode (opt-in marker file in the dispatcher module) validates full Apache syntax and routinely surfaces further errors the converter never looked at.
4. Re-review `/filters` and `/cache` by hand: the Cloud defaults are stricter (deny-by-default filters, `ignoreUrlParams` — [[dispatcher-ignoreurlparams]]) and some 6.x caching assumptions no longer hold behind the Adobe CDN.

## Traps

- "Converter output validated, so behaviour is unchanged" — false. Validation checks structure; `/allowedClients`, statistics-based load balancing and per-farm vhost logic are simply gone.
- On-premise mode with an idiosyncratic layout can produce a report longer than the config; it is often faster to start from the AEM Project Archetype dispatcher module and port rules across.
- Only **Apache 2.4** targets; Windows-style or IIS-adjacent configs are out of scope.
