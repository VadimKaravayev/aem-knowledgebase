# aio cloudmanager CLI plugin

Working reference for `@adobe/aio-cli-plugin-cloudmanager` — the CLI face of
the Cloud Manager API. Where the credential comes from (Developer Console
integration, product-profile-as-role) and the API/SDK family it belongs to
are in [[cloud-manager-api-cli-sdks]]; what the variables *mean* in
[[cloud-manager-variables]]; IMS credential types in
[[adobe-ims-authentication-types]].

---

## Install and auth

- `aio plugins:install @adobe/aio-cli-plugin-cloudmanager` (update via
  `aio plugins:update`). Needs the Adobe I/O CLI; Node ≥ 17 per the README, with odd
  (non-LTS) versions discouraged — practically, use an even LTS.
  Also runs standalone: `npm install -g` gives an
  `adobe-cloudmanager-cli` binary (config still via aio).
- **Interactive**: `aio auth:login` (browser IMS) + an org id via
  `aio cloudmanager:org:select` or `aio config:set cloudmanager_orgid <id>`.
- **Headless (CI)**: a service-account config JSON set as the IMS context
  `aio config:set ims.contexts.aio-cli-plugin-cloudmanager <file> --file
  --json`. **OAuth S2S** (`oauth_enabled: true`, `client_secrets` array) is
  the current form; the **JWT variant is dead** (removal announced for
  Jan 2025 — matches the IMS-wide JWT EOL) and legacy `jwt-auth` configs
  auto-migrate.
- Defaults to avoid retyping: `cloudmanager_programid`,
  `cloudmanager_environmentid` config keys.
- **`--permissions`** on any command prints which Cloud Manager product
  profiles may run it (e.g. advance → Business Owner, Deployment Manager,
  Program Manager) — the fast answer to "which role do I need for X".

## Command map (what exists, by noun)

- **org**: `org:list`, `org:select`.
- **program**: `list-programs`, `program:delete`, and the program-scoped
  lists — `list-environments`, `list-pipelines`, `list-current-executions`,
  `list-ip-allowlists`, `list-content-flows`, `list-content-sets`.
- **environment**: `set-variables`/`list-variables` (runtime env vars;
  `--variable`, `--secret`, `--delete`, per-service flags
  author/publish/preview, bulk via `--jsonFile/--jsonStdin/--yamlFile/
  --yamlStdin`), `delete`, `open-developer-console`,
  **`download-logs ENVID SERVICE NAME [DAYS]`** (default 1 day) and
  **`tail-log`** (live stream), `list-available-log-options`,
  IP-allowlist bind/unbind/list-bindings.
- **pipeline**: `update` (branch, repositoryId, tag, env ids), `delete`,
  `set-variables`/`list-variables` (build-time; per-service flags build /
  functionalTest / loadTest / uiTest), **`create-execution`**
  (`--emergency` for AMS), `invalidate-cache`, `list-executions`
  (`--limit`, default 20).
- **executions / quality gates**: `current-execution:get`,
  **`current-execution:advance`** (approve / override a gate),
  **`current-execution:cancel`** (reject), `execution:get-step-details`,
  `execution:get-step-log` (`-f sonarLogFile` for alternates),
  `execution:tail-step-log` (default action `build`),
  `execution:get-quality-gate-results` with actions **codeQuality,
  security, performance, contentAudit, experienceAudit**.
- **ip-allowlist**: create/update (`--cidr` required), delete, bind/unbind,
  `get-binding-details`.
- **content-flow / content-set** (Content Copy): `content-flow:create
  ENVID CONTENTSETID DESTENVID INCLUDEACL [TIER]` (tier default `author`),
  `get`, `cancel` (running only); `content-set:get`/`delete`.

All commands take `-j/--json` or `-y/--yaml` for scripting.

## Variables via stdin — the bulk/CI pattern

Both `set-variables` commands accept a JSON array
(`[{"name":…,"value":…,"type":"string"|"secretString"}]`) from file or
stdin; **an empty `value` deletes the variable**. This is the idempotent
"sync all variables from a file in CI" route, vs. one `--variable` flag
pair per value.

## Exit codes

`1` generic, `2` configuration, `3` flag/argument validation, `10` IMS
auth, `30` Cloud Manager API error — split your CI retry/alert logic on
10 vs 30 (credential problem vs API-side failure; 30 is what Adobe support
wants reported, with the request URL and response).

## References
- [adobe/aio-cli-plugin-cloudmanager (GitHub README)](https://github.com/adobe/aio-cli-plugin-cloudmanager/blob/main/README.md)

README-based (read Oct 2026), commands not individually re-run.
