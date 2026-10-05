# Guava (`com.google.common`) deprecation: finding and evicting every import

Adobe is fully deprecating the `com.google.common` (Guava) API on AEMaaCS, with removal scheduled. Cloud Manager flags it at deploy time: an **Actions Center** alert ("Deprecated Library Detected — com.google.common"), analyser warnings naming packages like `com.google.common.collect`, and a Code Quality step that pauses or fails. Affects AEMaaCS, AMS, and 6.5 LTS pipelines alike.

**The trap: it is almost never (only) your code.** The scanner inspects *all* OSGi imports — including transitive and test-scope dependencies — so a project with zero `com.google.common` references in `src/` still gets flagged. The usual carriers:

- outdated **ACS AEM Commons** (fix: upgrade to **≥ 6.11.0 with `classifier=cloud`**)
- **Core Components example packages** (`core.wcm.components.examples.*`) left in the `all` module — check the parent POM doesn't reintroduce them after you delete the explicit dep
- test libraries: `slf4j-test`, `io.wcm.testing.aem-mock.*`, sling-mock
- Asset Share Commons
- your own Guava utility usage

## Triage order

1. Locate carriers: `mvn dependency:tree -Dincludes=com.google.guava:*,com.google.common:*` plus `grep -R "com.google.common" ./src` for first-party usage. Confirm live imports in the OSGi Web Console (which bundle imports `com.google.common.*`).
2. Replace first-party usage with JDK equivalents or Apache Commons Collections 4.
3. Upgrade/remove the third-party carriers above.
4. If a stray import still leaks into the manifest, exclude `com.google.common.*` from `Import-Package` in the bundle plugin — and remember an explicit `Import-Package` pom entry survives code deletion (see [[osgi-import-package-version-range]] for that removal trap).
5. Redeploy via a full pipeline, then re-run `mvn dependency:tree` and check the analyser log; the Actions Center incident auto-resolves once clean.

## Seen in the wild: sandbox fails to dehibernate (Oct 2026)

A sandbox env wouldn't come out of hibernation; Adobe Support forced it and sent the launcher log. The log pins down how "deprecation" is actually enforced and when it bites:

- **Guava is not physically gone.** `com.adobe.granite.osgi.wrapper.guava` 15.0.0.0002 still ships in the `base-granite` feature — but its `com.google.common.*` exports live only in the **`com.adobe.aem.internal` API region**. The Sling Feature **API-Regions resolver hook** (`org.apache.sling.feature.apiregions.impl.ResolverHookImpl filterMatches`) removes it as a wiring candidate for any bundle whose requirement sits outside that region, so resolution fails with `missing requirement … com.google.common.collect; version>=15.0.0 !(version>=16.0.0)` even though the package demonstrably exists on the instance. For customer bundles, "deprecated" already behaves as "removed" at resolve time.
- **The failure surfaces at framework start, not at deploy.** Resolution happens when the bundle (re)starts — the env ran fine until hibernation; **dehibernation = restart = fresh resolution**, and the unresolvable bundle stalls the whole dehibernation. The deploy that introduced it succeeded. Same trigger class as the repoinit fresh-env trap in [[repoinit-acls-on-apps-and-libs]]: works everywhere except the environment that restarts.
- **Failures cascade.** `aem-guides-we-cafe.core` failed on `com.adobe.cq.commerce.core.components.models.productlist` — `[caused by: Unable to resolve core-cif-components-core … com.google.common.collect]`. Triage the root `caused by`, not the first ERROR line.
- **An unresolvable bundle does NOT block pipeline deploys** — after the forced dehibernation, a full Cloud Manager pipeline run against the same environment **passed**, customer bundles resolving and starting normally while the demo bundles stayed broken. So the readiness gating differs: deploy/rolling-switchover tolerates failed third-party bundles, dehibernation (at least in this incident) stalled on them. Don't conclude "env is fine" from a green pipeline, and don't conclude "deploys will fail" from a dehibernation stall.
- **The carrier was Adobe's own `aem-reference-demos-addon` (0.0.186): WKND 2.1.4 + CIF demo bundles**, not customer code — matching the doc's "it's almost never your code" rule. Support's "update the codebase" is off-target when the bundles come from the reference-demos add-on feature (`/opt/aem/launcher/cache/…` under feature `com.adobe.aem:aem-reference-demos-addon`); the fix there is updating/removing the demo add-on (or the WKND deployment, if self-deployed), not grepping your repo.

## 6.5 / on-prem wrinkle

Guava **cannot be installed as a standalone bundle** on 6.5/on-prem (`sling_bootstrap.txt` restrictions). If a dependency genuinely needs it there, shade/embed Guava inside your own bundle *without exporting* the Guava packages, and verify the bundle reaches Active. On AEMaaCS embedding is a stopgap at best — the API is headed for removal, so treat replacement as the real fix.

Related: [[aem-analyser-maven-plugin]] runs the same deprecated-API scan locally before Cloud Manager sees it; [[aem-sdk-java-ceiling-dual-65-cloud]] for the general shape of deprecation migrations blocked by a dual 6.5+Cloud baseline.

Source: Adobe KB [ka-34621](https://experienceleague.adobe.com/en/docs/experience-cloud-kcs/kbarticles/ka-34621) (updated Aug 2026); dehibernation incident verified from an Adobe Support launcher-log excerpt on a real AEMaaCS sandbox (AEM 6.6.0-8b7b8fbf1b base feature), Oct 2026.
