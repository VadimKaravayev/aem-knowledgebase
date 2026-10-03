# Cloud Manager FAQ: build and deploy answers

The durable nuggets from Adobe's Cloud Manager FAQ (AEMaaCS doc set) that no
other note carries — mostly build breakage after Java upgrades, versioning
rules, and the deploy-step failure checklist. Code-quality override roles are
in [[cloud-manager-code-quality-testing]]; `sling-distribution-importer`
mechanics in [[content-distribution-journal]]; API permissions in
[[cloud-manager-api-cli-sdks]].

---

## Java-upgrade build breakage (8 → 11)

- **Java 11 builds**: the FAQ's answer is the `maven-toolchains-plugin`.
  ⚠ Conflicts with the **AMS** build-environment page, where toolchains
  support was **removed in 2025.06.0**
  ([[cloud-manager-ams-build-environment]]) — on AMS use
  `.cloudmanager/java-version`; treat the toolchains answer as the
  AEMaaCS-side guidance.
- **`maven-scr-plugin` failure** after the switch: remove the plugin and
  convert Felix SCR annotations to **OSGi R6 annotations** — the error is a
  class-version mismatch (classes compiled by a newer JRE than the plugin
  supports).
- **`requireJavaVersion` enforcer failure**: **omit `requireJavaVersion`
  from `maven-enforcer-plugin`** — Cloud Manager *runs Maven with a
  different Java version than it compiles with*, so the enforcer check
  trips on the runner, not your target.

## Versioning rules

- **Dev deployments need `-SNAPSHOT`** at the end of the `pom.xml`
  `<version>` — that's what lets the same version re-install on subsequent
  deployments.
- **Stage/production: Cloud Manager generates the version itself** (and
  cuts git branches automatically). Keep a clean three-part version
  (`1.0.0`), increment per production release, and don't fight the
  generated suffix.

## Deploy-step failure checklist (in triage order)

1. **`sling-distribution-importer` permissions** — the most common cause:
   grant ACLs via a `RepositoryInitializer` repoinit config on the content
   paths it imports (**`/conf`, `/var`**, …). This FAQ answer confirms the
   exam-canonical fix from [[content-distribution-journal]]; Adobe's
   example config lives in the `cqsupport/cloud-manager` GitHub repo.
2. **Invalid OSGi configurations** breaking default services — read the
   deployment log.
3. **Invalid Dispatcher/Apache config** — reproduce with the local SDK
   Dispatcher Docker image.
4. **Content package replication failures** — simulate locally with an
   author+publish pair.

## Two more

- **`403 Forbidden` from `aio … set-variables`** = the user/service account
  lacks the **Deployment Manager** role (matrix in
  [[cloud-manager-api-cli-sdks]]).
- **Code-quality override**: Deployment Manager, Project Manager or
  Business Owner, from the pipeline results UI. ⚠ The FAQ says "all
  failures except **security rating** are non-critical" — looser than the
  metrics table, where **Reliability < D is also critical** on AEMaaCS
  ([[cloud-manager-code-quality-testing]]); trust the metrics table, but
  know the FAQ phrasing exists in circulating exam material.

## References
- [Cloud Manager FAQs (Adobe docs, AEMaaCS)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/faqs)
- [cqsupport/cloud-manager (GitHub)](https://github.com/cqsupport/cloud-manager) — Adobe support's troubleshooting snippets (build-step failures, importer repoinit example)

Docs-based (FAQ read Oct 2026), answers not reproduced.
