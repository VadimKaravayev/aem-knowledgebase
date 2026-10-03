# Cloud Manager Experience Audit

The **Google Lighthouse**-powered page audit built into Cloud Manager
(AEMaaCS). Pipeline step context in [[cloud-manager-pipeline-steps]]; the
CLI's `get-quality-gate-results` exposes it as the `experienceAudit` action
(and the older `contentAudit` name — the feature was renamed from Content
Audit) per [[aio-cloudmanager-cli]].

---

## Where and when it runs

- **Always on in every production Sites pipeline**, during the **Stage
  Testing** phase (after the stage deploy).
- **Optional** on full-stack and front-end **development pipelines** — an
  *Experience Audit* checkbox on the Source Code tab, paths on an
  Experience Audit tab.
- Also runnable **on demand** from the Reports tab (no pipeline): scans the
  latest ≤ 25 configured pages, finishes in minutes, refuses to start if
  the environment is deleted or another scan is pending.

## What it measures — and what it doesn't do

Four Lighthouse categories: **Performance, Accessibility, Best Practices,
SEO** (no PWA). Reported as the **median** score per configured page plus
the **delta against the previous scan**.

**It is informational only — it never blocks the pipeline.** That's the
exam-relevant contrast: the *performance/code-quality gates* can stop a
deployment ("important" vs critical failures,
[[cloud-manager-ams-production-pipelines]] for the behavior switch);
Experience Audit just shows the deployment manager the scores and the
trend. "Experience Audit failed the pipeline" is a distractor.

## Configuring pages

- Up to **25 paths**, each starting with `/`, **relative to the site**.
- **No paths configured → the site homepage is audited by default.**
- Production pipelines: set in the pipeline configuration; dev pipelines:
  the Experience Audit tab.

## Reading the results

- Pipeline execution page (Stage Testing) and the **Reports tab** both show
  them; the full report = **score trend** (filterable by category, page,
  time frame, trigger type) + per-scan results.
- **Slowest 5 pages** dialog with a **mobile/desktop toggle**.
- The Lighthouse Report column's date link downloads the **raw Lighthouse
  JSON**, viewable at `https://googlechrome.github.io/lighthouse/viewer/`
  — full per-audit detail beyond Cloud Manager's summary.
- Pages the audit couldn't reach are listed in an expandable error section;
  documented causes: access blocked by configuration, page doesn't exist,
  page needs non-basic authentication, or internal error. (An IP allow
  list or login wall on stage silently empties the audit.)

## Exam checklist

- Lighthouse under the hood; Stage Testing phase; production Sites
  pipelines always, dev pipelines opt-in.
- **Informational, never blocking** — unlike the performance/code-quality
  gates.
- ≤ **25 paths**, `/`-relative; **default = homepage only**.
- Median scores + delta vs previous; mobile/desktop views; raw Lighthouse
  JSON downloadable.

## References
- [Experience Audit (Adobe docs, Cloud Manager AEMaaCS)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/reports/report-experience-audit)

Docs-based (page read Oct 2026), not verified against a live pipeline.
