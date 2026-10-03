# Cloud Manager code quality testing

The code-quality gate's actual scoring model (AEMaaCS doc set; AMS
differences at the end). Pipeline
step context in [[cloud-manager-pipeline-steps]]; the Ask/Fail/Continue
switch for important failures in [[cloud-manager-ams-production-pipelines]];
gate results via CLI (`get-quality-gate-results`, action `codeQuality`) in
[[aio-cloudmanager-cli]].

---

## What scans

**SonarQube 9.9** (Java + AEM-specific rules, 100+ total) plus **OakPAL**
for content-package-level checks. The full rule list is downloadable from
Adobe (versioned, e.g. `2024-12-0` for the SonarQube 9.9 era).

## The three tiers

| Tier | Pipeline effect |
|---|---|
| **Critical** | immediate pipeline failure |
| **Important** | pipeline **pauses**; can be overridden and deployed in standard pipelines — **but NOT overridable in code-quality-only pipelines** |
| **Info** | informational, no effect |

The non-overridable-in-CQ-pipelines detail matters for PR-check setups: a
code-quality pipeline with an important finding simply fails.

## Metrics and thresholds (verbatim from the docs)

| Metric | Category | Fails when |
|---|---|---|
| **Security Rating** | Critical | **< B** |
| **Reliability Rating** | Critical | **< D** |
| **Maintainability Rating** | Important | **< A** |
| **Coverage** | Important | **< 50%** |
| Skipped Unit Tests | Info | > 1 |
| Open Issues | Info | > 0 |
| Duplicated Lines | Info | > 1% |
| Cloud Service Compatibility | Info | > 0 |

Rating scales: Security/Reliability A–E = no issues / ≥1 minor / ≥1 major /
≥1 critical / ≥1 blocker (vulnerability resp. bug). Maintainability =
remediation cost: A ≤5%, B 6–10%, C 11–20%, D 21–50%, E >50%.

**The asymmetry worth memorizing:** Security fails critical at **< B** (a
single *major* vulnerability kills the pipeline), while Reliability fails
critical only at **< D** (it takes a *blocker* bug) — security is held to a
far stricter bar than reliability. And Maintainability is Important at
**< A**: one code smell crossing the 5% remediation cost pauses the
pipeline for a human.

Mechanics: Coverage = `(CT + CF + LC) / (2*B + EL)` (conditions true/false,
lines covered / branches, executable lines); Duplicated Lines = ≥10
successive duplicate statements in Java (token/line-based for other
languages).

## False positives

Standard Java `@SuppressWarnings` with the Sonar rule id, at method or
class level: `@SuppressWarnings("squid:S2068")` (the hardcoded-password
rule is the canonical false-positive example, e.g. a property *name*
containing "password"). No special Adobe mechanism — plain Sonar
convention.

## AMS (AEM 6.x Cloud Manager) differences

Same engine (SonarQube 9.9 since 2025.2.0), same tiers, same false-positive
handling — but the AMS metrics table differs in exactly two places:

- **Reliability Rating is *Important* and fails at < C** (vs *Critical*,
  < D on AEMaaCS) — on AMS a major bug pauses the pipeline for an override;
  on cloud only a blocker bug fails it outright. A threshold question's
  answer depends on which platform the scenario names.
- **No Cloud Service Compatibility metric** (cloud-only by definition).

AMS also spells out the override roles: important failures are overridden
by the **deployment lead, project lead, or business owner**, override
subject to a timeout; code-quality-only pipelines allow no override on
either platform.

## Exam checklist

- SonarQube 9.9 + OakPAL; 100+ rules.
- Critical = fail now; Important = pause + override (except code-quality
  pipelines); Info = nothing.
- **Security < B critical, Reliability < D critical, Maintainability < A
  important, Coverage < 50% important** — everything else Info.
- One major vulnerability fails the build; it takes a blocker *bug* to do
  the same.
- False positives: `@SuppressWarnings("squid:<rule>")`.

## References
- [Code Quality Testing (Adobe docs, Cloud Manager AEMaaCS)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/test-results/code-quality-testing)
- [Code Quality Testing (Adobe docs, Cloud Manager for AMS)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-manager/content/using/code-quality-testing)

Docs-based (page read twice Oct 2026, metrics table cross-checked), not
reproduced against a live pipeline.
