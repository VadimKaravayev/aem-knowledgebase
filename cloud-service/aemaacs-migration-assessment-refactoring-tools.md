# Migration to AEMaaCS: Best Practices Analyzer and Index Converter

The two tools that come **before** the Content Transfer Tool in Adobe's
migration journey. The **Best Practices Analyzer (BPA)** tells you how far
your 6.x implementation is from cloud-ready and sizes the refactoring; the
**Index Converter** is one of the code refactoring tools that fixes a specific
finding, custom Oak index definitions that the cloud will not accept.

```
assess ──▶ refactor ──▶ transfer content
 BPA        Index Converter,        CTT ([[content-transfer-tool]])
            repository restructuring,
            dispatcher/code converters
```

Companion notes: [[oak-indexing-aemaacs]] (what a cloud-compatible index
definition looks like), [[content-transfer-tool]], [[cloud-manager-api-cli-sdks]]
(the same `aio` CLI hosts the migration plugin).

---

## 1. Best Practices Analyzer (BPA)

A package you install on the **existing 6.x instance**. It scans the
implementation and produces a report of everything that deviates from AEM
best practice, with links to the implication and the fix for each category.
Adobe positions it as **the first step of the transition** and as the input
for **estimating time and cost**: it gathers what you would otherwise collect
by hand.

Report categories:

| Category | Typical findings |
|---|---|
| Application functionality that must be **refactored** | unsupported APIs, custom login modules, long-running jobs in request threads |
| Repository items in **unsupported locations** | content under `/etc`, `/apps` overlays that should be `/conf` or `/content` |
| **Legacy UI** dialogs and components to modernise | Classic UI dialogs, foundation components |
| **Deployment and configuration** issues | run-mode configs in the wrong place, embedded packages, OSGi configs needing secrets |
| **6.x features replaced or unsupported** on AEMaaCS | replication agents, custom workflows on removed models, Communities, Screens classic, etc. |

- Run it on author (production or a fresh clone of it), download the report
  from the instance, and treat the category counts as the refactoring backlog.
- It assesses readiness; it does not fix anything and it does not move
  content. Findings map to the code refactoring tools and to the
  repository restructuring guides.

## 2. Index Converter

Converts **custom Oak index definitions** into AEMaaCS-compatible custom
index definitions, ready to ship under `/apps` in the code base.

What it handles, and what it does not:

- Only **`lucene`-type** custom definitions found under **`/apps`** (from any
  content package) or directly under **`/oak:index`**.
- **Does not** convert lucene indexes defined on **`nt:base`**.
- **Ensure Oak Index** (ACS Commons) definitions are **not supported on
  AEMaaCS at all**. Convert them to plain Oak index definitions first, then
  run the converter. The manual rules for that conversion:
  1. skip any Ensure Definition whose `ignore` property is `true`;
  2. set `jcr:primaryType` to `oak:QueryIndexDefinition`;
  3. drop the properties the Ensure OSGi configuration lists as ignored;
  4. remove the `/facets/jcr:content` subtree.

How to run it:

| Form | When |
|---|---|
| **`aio-cli-plugin-aem-cloud-service-migration`** (Adobe I/O CLI plugin, Adobe's recommended route) | as part of the whole code refactoring pass; the same plugin hosts the dispatcher converter and repository modernizer |
| **`aem-cs-source-migration-index-converter`** standalone | when you only need the index step |

Both are on GitHub under `adobe`.

Why it matters beyond compliance: on AEMaaCS custom indexes are deployed as
code and rebuilt by the platform, and CTT does not copy indexes, so any
definition that the converter cannot produce simply will not exist on the
target ([[oak-indexing-aemaacs]], [[content-transfer-tool]]).

## Exam checklist

- **BPA** = readiness report and cost estimate, installed on the **source**
  6.x instance; five categories; first step of the journey.
- **Index Converter** = lucene custom indexes under `/apps` or `/oak:index`
  → cloud format; **not `nt:base`** indexes; **Ensure Definitions must be
  converted manually first** and are unsupported in the cloud.
- Run via the **aio migration plugin** (recommended) or standalone.
- Order: BPA → refactor (Index Converter, restructuring) → CTT.

## References
- [Best Practices Analyzer overview (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/migration-journey/cloud-migration/best-practices-analyzer/overview-best-practices-analyzer)
- [Index Converter (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/migration-journey/refactoring-tools/index-converter)
- [aio-cli-plugin-aem-cloud-service-migration (GitHub)](https://github.com/adobe/aio-cli-plugin-aem-cloud-service-migration)
- [aem-cs-source-migration-index-converter (GitHub)](https://github.com/adobe/aem-cloud-service-source-migration/tree/master/packages/index-converter)
