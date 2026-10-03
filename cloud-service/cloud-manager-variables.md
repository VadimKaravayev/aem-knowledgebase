# Cloud Manager variables — build-time vs pipeline vs runtime environment

Three distinct variable mechanisms that exam questions (and real setups) routinely conflate. Rule of thumb: **`CM_*` = build-time facts Cloud Manager gives you; pipeline variables = build-time values you give it; environment variables = runtime values for OSGi/Dispatcher.**

## 1. Variables Cloud Manager sets in the build environment

Available as environment variables during the Maven build (Adobe "Build Environment" docs):

| Variable | Holds |
|---|---|
| `CM_BUILD` | Always set when the build runs in Cloud Manager — the standard "am I in CM?" switch |
| `CM_PROGRAM_ID` / `CM_PROGRAM_NAME` | Program being built |
| `CM_PIPELINE_ID` / `CM_PIPELINE_NAME` | Pipeline that triggered the build |
| `CM_AEM_PRODUCT_VERSION` | AEM release version the pipeline targets |
| `ARTIFACTS_VERSION` | Auto-generated version Cloud Manager stamps on the artifacts (stage/prod pipelines) |

The two Maven profile-activation idioms to recognize:

```xml
<activation><property><name>env.CM_BUILD</name></property></activation>   <!-- only INSIDE Cloud Manager -->
<activation><property><name>!env.CM_BUILD</name></property></activation>  <!-- only OUTSIDE (local dev) -->
```

Maven resolves environment variables only under the `env.` prefix — something like `STAGE.CM_BUILD` names a property that never exists, so the profile never activates. **Exam trap:** the inverted (`!`) form is the standard pattern for local-only profiles (e.g. local deployment via `autoInstallPackage`), which makes it easy to copy into a profile that was meant to run *inside* Cloud Manager — symptom is "expected build output missing from the pipeline run but present locally" (see `devops/questions/question-31.md` in the aem-architect-lab repo).

## 2. Custom pipeline variables (build-time, you set them)

Added per pipeline via CLI/API, not the UI:

```
aio cloudmanager:set-pipeline-variables PIPELINEID --variable MY_VAR value --secret MY_SECRET value
```

Exposed as environment variables in the build container; `--secret` values are masked in build logs. Typical use: credentials for a private/password-protected Maven repository referenced from the build. They exist **only in the build environment**, never at runtime.

## 3. Runtime environment variables (per environment, for OSGi/Dispatcher)

A separate feature entirely: set per *environment* in Cloud Manager (Environment Configuration UI, or `aio cloudmanager:set-environment-variables`). Two flavors, consumed from OSGi configs as:

- standard: `$[env:MY_VAR]` (optionally with `;default=...`)
- secret: `$[secret:MY_SECRET]` — secrets are stored separately, not readable back from the UI/API

This is the supported way to keep per-environment values and credentials out of Git. Constraints: names are uppercase letters/digits/underscore, must not start with reserved prefixes (`INTERNAL_`, `ADOBE_`); changes apply without a pipeline run (short propagation delay). Dispatcher configs can also reference them. Exam angle: secrets belong in `$[secret:...]` via Cloud Manager — never hardcoded in repo OSGi configs; and don't confuse these with run modes, which select *which* config file applies while variables fill in *values* inside it.

Related: [cloud-manager-api-cli-sdks.md](cloud-manager-api-cli-sdks.md) (the `aio cloudmanager` plugin used to set both kinds), [cloud-manager-pipeline-steps.md](cloud-manager-pipeline-steps.md) (where in the pipeline the build environment exists).

From Adobe build-environment and environment-variables docs via exam prep, Oct 2026 — documented, not screen-verified.
