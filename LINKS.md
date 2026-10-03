# Links

Registry of useful external links — repos, tools, doc hubs — that don't fit a
single topic note. One line each: link + what it's for + the KB note that
covers it in depth, if one exists. Add here first; promote to a full note
only when there's durable insight to record.

## Repos and CLI tools

- [adobe/aio-cli-plugin-cloudmanager](https://github.com/adobe/aio-cli-plugin-cloudmanager) — Cloud Manager CLI plugin for the Adobe I/O CLI (pipelines, executions, variables, logs, IP allowlists, content flows). Note: [aio-cloudmanager-cli.md](cloud-service/aio-cloudmanager-cli.md).
- [adobe/aio-lib-cloudmanager](https://github.com/adobe/aio-lib-cloudmanager) — the Node.js SDK the CLI plugin is built on; App Builder-friendly. Covered in [cloud-manager-api-cli-sdks.md](cloud-service/cloud-manager-api-cli-sdks.md).
- [adobe/aio-cli-plugin-aem-cloud-service-migration](https://github.com/adobe/aio-cli-plugin-aem-cloud-service-migration) — code-refactoring/migration tooling (Index Converter et al.) for AEMaaCS moves. Covered in [aemaacs-migration-assessment-refactoring-tools.md](cloud-service/aemaacs-migration-assessment-refactoring-tools.md).
- [Cloud Manager for AEM GitHub App](https://github.com/apps/cloud-manager-for-aem) — the app a GitHub org owner installs for BYO-GitHub pipelines. Covered in [cloud-manager-git-repositories.md](cloud-service/cloud-manager-git-repositories.md).
- [adobe/aem-project-archetype](https://github.com/adobe/aem-project-archetype) — the Maven archetype AEM projects start from (6.5 and AEMaaCS). Its [ui.tests module](https://github.com/adobe/aem-project-archetype/tree/master/src/main/archetype/ui.tests) is the Dockerized UI-test skeleton Cloud Manager runs; covered in [cloud-manager-functional-ui-testing.md](cloud-service/cloud-manager-functional-ui-testing.md).
- [adobe/aem-test-samples (aem-cloud branch)](https://github.com/adobe/aem-test-samples/blob/aem-cloud/README.md) — Adobe's sample functional-test suite (smoke/ReplicationIT) and the template for `it.tests`; run against cloud envs with `-Peaas-local -Dcloud.author.url=…`. Covered in [cloud-manager-functional-ui-testing.md](cloud-service/cloud-manager-functional-ui-testing.md).
- [adobe/aemanalyser-maven-plugin](https://github.com/adobe/aemanalyser-maven-plugin/) — runs Cloud Manager's build analyzers locally (`aem-analyse` goal). Note: [aem-analyser-maven-plugin.md](cloud-service/aem-analyser-maven-plugin.md).
- [adobe/aem-guides-wknd](https://github.com/adobe/aem-guides-wknd) — WKND reference site; releases ship cloud and `-classic` (6.5) zips. Install gotchas in [wknd-onprem-install.md](infrastructure-ops/wknd-onprem-install.md).

## Doc hubs

- [wcm.io How-to articles](https://wcm-io.atlassian.net/wiki/spaces/WCMIO/pages/1161232394/How-to+articles) — ~21 practical guides (FileVault validation compliance, Java 11 switch, clientlib proxy mode, AEM Mocks → JUnit 5, CONGA for AEMaaCS, content-package plugin migration…). Confluence truncates in WebFetch — read via REST API: `https://wcm-io.atlassian.net/wiki/rest/api/content/<pageId>?expand=body.storage`. Captured so far: [filevault-validation-compliance.md](infrastructure-ops/filevault-validation-compliance.md).

- [Cloud Manager API reference](https://developer.adobe.com/experience-cloud/cloud-manager/reference/api) (hub: [developer.adobe.com/experience-cloud/cloud-manager](https://developer.adobe.com/experience-cloud/cloud-manager/)) — HAL API + events; covered in [cloud-manager-api-cli-sdks.md](cloud-service/cloud-manager-api-cli-sdks.md). The reference UI is a JS shell WebFetch can't read; the **raw OpenAPI specs** live in [AdobeDocs/cloudmanager-api-docs](https://github.com/AdobeDocs/cloudmanager-api-docs) at `static/api.yaml`, `static/events.yaml`, `static/models.yaml` (raw.githubusercontent.com works; the docs-site `/experience-cloud/cloud-manager/api.yaml` path 404s).
- [Cloud Manager CLI and SDKs](https://developer.adobe.com/experience-cloud/cloud-manager/cli-and-sdks/) — official landing page for the `aio` CLI plugin, Node.js SDK, Java SDK and the pipeline-triggering GitHub Action. Working CLI reference: [aio-cloudmanager-cli.md](cloud-service/aio-cloudmanager-cli.md); tool family: [cloud-manager-api-cli-sdks.md](cloud-service/cloud-manager-api-cli-sdks.md).
- [AEM 6.5 developer best practices hub](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/implementing/developing/bestpractices/best-practices) — link hub only (no substance of its own); its sub-pages are candidates for notes.
- [Git Integration with Cloud Manager (AMS)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-manager/content/managing-code/git-integration) — video-tutorial hub for the five customer-git ↔ Adobe-git sync scenarios (initial sync, pipeline-aligned branching, feature branches, release sync, tags back); summarized in [cloud-manager-git-repositories.md](cloud-service/cloud-manager-git-repositories.md).
