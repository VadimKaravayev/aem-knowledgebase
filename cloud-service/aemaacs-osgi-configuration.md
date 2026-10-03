# OSGi configuration on AEMaaCS

The cloud-specific OSGi configuration contract: format, the inline-vs-variable
decision, preview-tier inheritance, local-dev secrets, and how variable
deployments behave. Replace-not-merge run-mode resolution and the 6.5 drift
story are in [[osgi-config-runmode-resolution]]; the three variable
*mechanisms* side by side in [[cloud-manager-variables]]; the closed run-mode
set and `getRunModes()` runtime gotcha in
[[rolling-deployment-two-version-overlap]]; CLI command forms in
[[aio-cloudmanager-cli]].

---

## Format and placement

- **`.cfg.json` only** — `.cfg`, `.config` and XML `sling:OsgiConfig` are
  superseded. Factory configs are `<factoryPID>~<name>.cfg.json` (the OSGi R7
  `~` separator; the older `-` separator belongs to the classic formats —
  era split in [[osgi-config-runmode-resolution]]).
- Files live in the **`ui.config`** package under
  `/apps/<project>/osgiconfig/config.<runmode>`. On a running cloud
  environment they are **not browsable under `/apps`** — view them via the
  Developer Console instead.
- **Preview has no config folder.** `config.preview` cannot be declared;
  the **preview tier inherits the publish tier's OSGi configuration**. The
  only way to vary a value on preview is an environment variable bound to
  the preview service (below).

## The decision ladder: inline → env var → secret

1. **Inline values are the standard** — versioned in Git, implicitly tied
   to the code deployment that uses them.
2. **`$[env:NAME]`** (optionally `$[env:NAME;default=<value>]`) only when a
   non-secret value genuinely varies **across dev environments or on the
   preview tier**. Adobe explicitly recommends **against** non-secret env
   vars on stage/production — those values belong inline, per run mode.
3. **`$[secret:NAME]`** for anything that can't sit in Git; appropriate on
   every tier including production.

One file may mix all three. **Placeholders do not work in repoinit
statements** — a repoinit script cannot consume env vars or secrets, so
anything environment-variable-shaped can't be provisioned via repoinit.

## Variable rules (environment variables / secrets)

- Name: 2–100 chars, regex `[a-zA-Z_][a-zA-Z_0-9]*`; value ≤ **2048 chars**;
  **≤ 200 variables per environment**.
- Reserved prefixes (customer-set ones are ignored): `INTERNAL_`, `ADOBE_`,
  **`CONST_`**; **`AEM_` is the product's public prefix** for
  Adobe-defined-but-customer-settable switches.
- Per-tier values: one variable name can carry different values for
  **author / publish / preview** via the API/CLI `service` parameter —
  Adobe's preferred shape over `author_X`/`publish_X` name prefixes.
- Set via Cloud Manager UI, API (`PATCH …/variables`, type `string` |
  `secretString`, **empty value = delete**) or CLI
  (`aio cloudmanager:environment:set-variables` with
  `--variable/--secret/--delete`). API caller needs the **Deployment
  Manager** role.

## How a variable deployment behaves (the part people miss)

- Setting variables **restarts author and publish** and takes effect in
  minutes — a mini-deployment, **but the quality gates and tests of a real
  pipeline are skipped**.
- The call **can fail while any pipeline (customer or AEM update) is
  running**, depending on its phase — don't wire it blindly into CI that
  races the deploy.
- Order: **set variables before deploying the code that reads them.**
- **Variables are not versioned** — a code rollback keeps the *new*
  variable values, the same shape as "rollback restores code, not content"
  ([[rolling-deployment-two-version-overlap]]). Adobe's mitigation is the
  **additive strategy**: introduce a *new* variable name rather than
  repointing an old one, so old code never sees the new value; delete the
  old name only days later.

## Local development

- Non-secret: plain process env vars (`export NAME=value`; direnv helps).
- **Secrets are read from files, not env vars**: set
  `org.apache.felix.configadmin.plugin.interpolation.secretsdir=${sling.home}/secretsdir`
  in `crx-quickstart/conf/sling.properties`, then one file per secret whose
  **filename is exactly the placeholder name with no extension**
  (`server_password`, not `server_password.txt` — the extension is the
  classic reason the placeholder resolves to nothing).

## Authoring and verifying configs

- **Web-console recipe (local SDK only)**: configure the component in
  `/system/console` → note the PID → **OSGi Installer Configuration
  Printer** with serialization format **"OSGi Configurator JSON"** → paste
  the emitted JSON into `<PID>.cfg.json` in `ui.config`. Caveat: the local
  console *writes* `.cfg.json` into the repository as a side effect —
  surprising state during local dev. On cloud environments the console
  doesn't exist and the Developer Console is **read-only**.
- **Verify on cloud**: Developer Console → pick the pod → Status tab →
  Status Dump **Configurations** → find the `pid`, inspect `properties`,
  diff against the run-mode folder you expect to have won.

## Exam checklist

- `.cfg.json` only; factory `~name`; configs in `ui.config`, not visible
  under `/apps` on cloud.
- **Preview inherits publish OSGi config** — no `config.preview`; vary
  preview via a service-bound variable.
- Inline is the default; non-secret env vars ≈ dev/preview only; secrets
  anywhere.
- **No placeholders in repoinit.**
- Variables: 2–100-char names, ≤ 2048-char values, ≤ 200/env, reserved
  `INTERNAL_/ADOBE_/CONST_`, public `AEM_`; set them **before** the code
  deploy; restart in minutes, **no quality gates**; not versioned → additive
  renames.
- Local secrets = extension-less files in `secretsdir`.

## References
- [Configuring OSGi for AEM as a Cloud Service (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/deploying/configuring-osgi)

Docs-based (page read Oct 2026), not reproduced on a live environment.
