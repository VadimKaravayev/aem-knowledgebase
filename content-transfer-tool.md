# Content Transfer Tool (CTT)

Adobe's tool for migrating repository **content** (not code) from an on-prem/AMS AEM instance into AEM as a Cloud Service.

## What it does

Extracts JCR content (pages, DAM assets, users/groups, etc.) from the source instance into a portable **migration set**, which is then ingested into the target AEMaaCS environment via Cloud Manager. Code migration is a separate, unrelated workstream — see below.

## Prerequisites

- **Minimum source version: AEM 6.3+.** CTT cannot be installed/run against anything older (e.g. 6.2) — the instance must be upgraded first. This is a hard version gate, not a "nice to have."
- Installing the *latest service pack* is not itself required — only a *supported version* is. (Often bundled into an upgrade anyway, but not the CTT requirement per se.)

## Prep for large repositories

- **Review total index size before migrating.** Indexes are not transferred as content — they are rebuilt on the target. Oversized or unused custom indexes inflate extraction time, migration-set size, and post-ingestion reindexing. **Prune** them: identify indexes no longer referenced by any query and delete/disable those definitions before running CTT, so only indexes actually in use get rebuilt on the target.

## Scope boundary (common exam trap)

CTT prep questions are sometimes mixed with the separate requirement to refactor application code for AEMaaCS (splitting into mutable/immutable packages, removing unsupported APIs — handled by the Cloud Manager Code Refactoring Tools). That refactor is real and necessary for the overall migration, but it is not a CTT-specific requirement — CTT only cares about content extraction/ingestion.

## References
- Question 17 in this repo (`devops/questions/question-17.md`) — AEM 6.2 large-repo migration scenario this note was written from.
