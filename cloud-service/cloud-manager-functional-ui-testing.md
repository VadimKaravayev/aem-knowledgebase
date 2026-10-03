# Cloud Manager functional and UI testing

The *blocking* test gates of AEMaaCS pipelines — product functional, custom
functional, custom UI — and how they differ. Step order in
[[cloud-manager-pipeline-steps]]; the deliberately **non-blocking** contrast
is [[cloud-manager-experience-audit]]; the static-analysis gate is
[[cloud-manager-code-quality-testing]].

---

## The three test types

| Type | Who writes it | Tech | Module | Timeout |
|---|---|---|---|---|
| **Product functional tests** | Adobe (not customizable) | HTTP integration tests: JUnit + Maven + AEM Testing Clients | — | — |
| **Custom functional tests** | You | same stack (JUnit + aem-testing-clients) | **`it.tests`** (archetype) | **60 min** |
| **Custom UI tests** | You | **Docker container**: Cypress, Playwright, Selenium, Java or JavaScript | **`ui.tests`** (archetype) | **30 min** |

- Product functional tests exist to stop *your* code from breaking *core*
  AEM functionality — they run always and you can't opt out or extend them.
- Custom functional tests are plain HTTP integration tests against the
  deployed instances; Adobe's guidance is to keep the suite at **~15
  minutes or less**, covering key features and primary user flows, not
  exhaustive regression.
- Custom UI tests ship as a Docker image built from `ui.tests`;
  **non-Selenium frameworks run through an HTTP proxy** in the test
  container setup.

## When they run, and whether they block

| Gate | Production pipeline | Non-production pipeline | Blocking |
|---|---|---|---|
| Unit tests | always | always | yes |
| Product functional | always | always | yes |
| Custom functional | always | **opt-in** | yes (60 m timeout) |
| Custom UI | always | **opt-in** | yes (30 m timeout) |

All four **block the deployment on failure** — the clean contrast with
Experience Audit, which never does. In a production pipeline they run in
the stage-testing phase against the freshly deployed stage environment.

## Exam checklist

- Product functional tests: Adobe-maintained, always on, not customizable
  — "disable the product tests" is never an answer.
- Custom functional = `it.tests`, JUnit/aem-testing-clients, 60-min cap;
  custom UI = `ui.tests`, Dockerized Cypress/Playwright/Selenium, 30-min
  cap; both always-on in production, **opt-in in non-prod pipelines**.
- All functional/UI gates **block**; Experience Audit doesn't.
- Suite sizing guidance: ~15 min, key flows only.

## Writing and running them — the reference repos

**Functional (HTTP) tests — `adobe/aem-test-samples`, branch `aem-cloud`:**
Adobe's sample suite (a `smoke` module: CreatePageAdminIT,
CreatePageAsAuthorUserIT, ReplicationIT…) and the de-facto template for
`it.tests`. Key mechanics:

- The build produces a **`jar-with-dependencies`** — the standalone-runnable
  artifact shape Cloud Manager picks up.
- Local run: `mvn clean verify -Ptest-all`. Against a real AEMaaCS env:
  `mvn -Peaas-local clean verify -Dcloud.author.url=… -Dcloud.author.user=admin
  -Dcloud.author.password=… -Dcloud.publish.url=… -Dcloud.publish.user=…
  -Dcloud.publish.password=…`.
- Under the hood that profile sets the Sling testing properties:
  `sling.it.instances=2`, and per instance
  `sling.it.instance.{url,runmode,adminUser,adminPassword}.{1,2}`
  (1 = author, 2 = publish), plus
  `sling.it.configure.default.replication.agents=false`.
- Tests assume an **admin-equivalent user** (creates content, users,
  groups, replicates) and working author→publish replication;
  `ReplicationIT` creates/deletes a randomized page
  (`/content/test-site/testpage_<uuid>`) — i.e. the suite writes to the
  target, don't point it at production casually.

**UI tests — the archetype's `ui.tests` module:**

- Structure: your code in **`test-module/`**; `pom.xml`, `Dockerfile`,
  `wait-for-grid.sh`, compose files and the assembly descriptor are
  **protected — don't modify** (Cloud Manager's contract with the image).
  Reports land in `target/reports`.
- Local run: `mvn verify -Pui-tests-local-execution` with env vars
  `AEM_AUTHOR_URL/USERNAME/PASSWORD` (defaults localhost:4502/admin/admin),
  `AEM_PUBLISH_*`, `SELENIUM_BROWSER` (`chrome` default, or `firefox`),
  `HEADLESS_BROWSER` (default `false`).
- Docker image build: `mvn clean install -Pui-tests-docker-build`; from
  inside the container the local author is
  `http://host.docker.internal:4502`.
- The sample repo also carries per-framework variants (`ui-cypress`,
  `ui-playwright`, `ui-wdio`, `ui-selenium-webdriver`) to start from.

## References
- [Functional Testing (Adobe docs, Cloud Manager AEMaaCS)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/test-results/functional-testing/functional-testing)
- [adobe/aem-test-samples, aem-cloud branch (GitHub)](https://github.com/adobe/aem-test-samples/blob/aem-cloud/README.md)
- [aem-project-archetype ui.tests module (GitHub)](https://github.com/adobe/aem-project-archetype/tree/master/src/main/archetype/ui.tests)

Docs-based (overview page read Oct 2026), not reproduced against a live
pipeline.
