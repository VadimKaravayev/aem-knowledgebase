# Content Transfer Tool (CTT)

Adobe's tool for migrating repository **content** (not code) from an on-prem
or AMS AEM instance into AEM as a Cloud Service. It runs inside **Cloud
Acceleration Manager (CAM)**, not Cloud Manager: CAM holds the project, the
migration sets, the ingestion logs and the validation reports.

Companion notes: [[aemaacs-migration-assessment-refactoring-tools]] (Best
Practices Analyzer and Index Converter, the steps *before* CTT),
[[oak-indexing-aemaacs]] (why indexes are rebuilt rather than copied),
[[cloud-manager-program-types]] (the target environments).

---

## What it does

Extracts JCR content (pages, DAM assets, users and **groups**, which are
migrated automatically) from the source instance into a **migration set**,
then ingests that set into one or more target AEMaaCS environments. Code
migration is a separate, unrelated workstream; see the scope boundary below.

Because CTT is integrated with CAM you get:

- **extract once, ingest into multiple environments in parallel** (dev, stage,
  prod from the same set),
- persisted **ingestion logs** for troubleshooting,
- **Validation** and **Principal migration** reports,
- guardrails, loading states and error handling in the UI.

## Two phases

```
source AEM ──extraction──▶ migration set (Adobe cloud storage) ──ingestion──▶ AEMaaCS env(s)
```

| Phase | What happens | Top-up switch |
|---|---|---|
| **Extraction** | content copied from source into the migration set, a temporary Adobe-provided cloud storage area | **overwrite must be disabled** to add a delta to an existing set |
| **Ingestion** | migration set applied to the target environment | **wipe must be disabled** to layer the delta on existing content |

## Migration set attributes

- **Max ten migration sets per CAM project.** Unique names.
- **Differential top-up**: after the first full transfer, transfer only what
  changed since the last run. Adobe's recommendation is **frequent top-ups**
  so the final content freeze before go-live is short.
- **Expiry after roughly 45 days of inactivity.** Warning indicators appear on
  the project card and the migration job rows first; after expiry the data is
  gone. Any of these actions resets the clock: editing the description,
  fetching the extraction key, running an extraction to it, running an
  ingestion from it.

## Prerequisites and hard limits

| Consideration | Supported today | Above the limit |
|---|---|---|
| **Source AEM version** | **6.3 or higher** | upgrade first; this is a hard gate, and the latest service pack is *not* itself required |
| **Segment store** | < **750 million** JCR nodes; ≤ **500 GB** online-compacted on **Author**, ≤ **50 GB** on **Publish** | Customer Care ticket |
| **Total repository** (segment + data store) | ≤ **20 TB** for **File Data Store** | Customer Care ticket; with the optional **pre-copy** step, **S3 and Azure data stores may exceed 20 TB** |
| **Total Lucene index size** | ≤ **25 GB**, excluding `/oak:index/lucene` and `/oak:index/damAssetLucene` | Customer Care ticket |
| **Immutable paths** | not migratable; selected `/etc` paths only for **Forms → Forms as a Cloud Service** | restructure per Common Repository Restructuring first |
| **Property values (MongoDB target)** | ≤ **16 MB** per node property, enforced by MongoDB; ingestion fails otherwise | run Adobe's `oak-run` script before extraction, convert oversize values to **binaries** |

Notes on the table:

- The optional **pre-copy** step is what makes very large repositories
  practical. It applies to File Data Store, Amazon S3 and Azure data stores
  and significantly speeds up the transfer.
- The 16 MB rule exists because AEMaaCS author runs on a document store
  (MongoDB); a segment-store source never enforced it, so it only surfaces at
  ingestion unless you scan first.

## Prep for large repositories

- **Review total index size before migrating.** Indexes are not transferred
  as content; they are rebuilt on the target. Oversized or unused custom
  indexes inflate extraction time, migration-set size and post-ingestion
  reindexing. **Prune** definitions no query references. Then run them
  through the **Index Converter** so they are cloud-compatible
  ([[aemaacs-migration-assessment-refactoring-tools]]).
- Plan a **first full transfer early**, then top-ups; the last top-up during
  the freeze should be small.

## Scope boundary (common exam trap)

CTT prep questions are sometimes mixed with the separate requirement to
refactor application code for AEMaaCS (mutable/immutable package split,
unsupported APIs, index definitions). That refactor is real and necessary for
the migration, handled by the Best Practices Analyzer and the code refactoring
tools, but it is not a CTT requirement. CTT only cares about content
extraction and ingestion.

## Exam checklist

- CTT lives in **Cloud Acceleration Manager**; content only, groups included.
- Source ≥ **6.3**; 750 M nodes / 500 GB author / 50 GB publish; 20 TB FDS
  (more with pre-copy on S3/Azure); 25 GB Lucene; 16 MB property cap.
- **Ten** migration sets per project; **45-day** inactivity expiry, reset by
  touching the set.
- Top-up = extraction with **overwrite off** + ingestion with **wipe off**.
- Extract once, **ingest to many environments in parallel**.
- Immutable paths do not transfer; only Forms gets `/etc` exceptions.

## References
- [Content Transfer Tool overview (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/migration-journey/cloud-migration/content-transfer-tool/overview-content-transfer-tool)
- [Prerequisites for Content Transfer Tool (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/migration-journey/cloud-migration/content-transfer-tool/prerequisites-content-transfer-tool)
- Question 17 in this repo (`devops/questions/question-17.md`), the AEM 6.2 large-repo migration scenario this note was first written from.
