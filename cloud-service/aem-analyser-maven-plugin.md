# AEM Analyser Maven Plugin

`com.adobe.aem:aemanalyser-maven-plugin` (goal `analyse`) — runs **the same
analyzers Cloud Manager's build step runs**, locally at build time, so
deploy-blocking findings surface before a pipeline ever runs (the
"shift-left" tool [[aemaacs-sdk]] alludes to; pipeline context in
[[cloud-manager-pipeline-steps]]). Source:
github.com/adobe/aemanalyser-maven-plugin; archetype projects ship it
preconfigured via the `aemanalyser.version` property.

---

## Behaviors worth knowing

- **Local ⇄ Cloud Manager parity is explicit**: Adobe's matrix marks every
  analyzer as running in both. A clean local `analyse` ≈ a clean Build
  Images analysis (not a substitute for runtime testing — the SDK ≠ cloud
  runtime).
- **It analyses against the *latest* SDK, not the project's configured
  version** — findings can change without any code change when Adobe
  releases a new SDK; that's a feature (matches what Cloud Manager will
  judge you by), not nondeterminism in your build.
- Minimum usable version **1.1.2** — older versions die with
  `arraycopy: source index -1 out of bounds for char[65536]`; that error
  = bump the plugin, not a project problem.

## What the analyzers catch (grouped)

**OSGi resolution** — `api-regions-exportsimports` (every Import-Package
satisfied by some Export-Package in the deployment), 
`bundle-unversioned-packages` (imports/exports must carry versions),
`requirements-capabilities` (bundle req/cap matching),
`bundle-nativecode` (no native code), `bundle-resources`
(Sling-Bundle-Resources headers break on clustered/cloud),
`artifact-rules` (bundle/package dependency validity).

**API-surface discipline** — `api-regions` (+ check-order, dependencies,
duplicates), **`api-regions-crossfeature-dups`** (your bundle may not
export packages that shadow AEM's public API), `region-deprecated-api`
(deprecated-API usage — the analyzer behind "removal 2027-xx" warnings),
**`aem-provider-type`** (you may not *implement* `@ProviderType`
interfaces).

**Configuration & content** — `repoinit` (syntax of repoinit scripts),
`configuration-api` + `configurations-basic` (OSGi config restrictions and
common mistakes), `aem-env-var` (variable naming rules,
[[aemaacs-osgi-configuration]]), `content-package-validation` (FileVault
validators; `jackrabbit-docviewparser` on by default — malformed
`.content.xml` fails here).

## Exam checklist

- Same checks locally and in Cloud Manager; runs against the **latest
  SDK** regardless of project config.
- Catches: unresolvable/unversioned OSGi imports, shadowing AEM public
  API, implementing ProviderType interfaces, deprecated-API usage, repoinit
  syntax, env-var naming, malformed content packages.
- `arraycopy … out of bounds` = plugin < 1.1.2, upgrade.

## References
- [Build Analyzer Maven Plugin (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-core-components/using/developing/archetype/build-analyzer-maven-plugin)
- [adobe/aemanalyser-maven-plugin (GitHub)](https://github.com/adobe/aemanalyser-maven-plugin/)

Docs-based (page read Oct 2026), analyzer list as documented there.
