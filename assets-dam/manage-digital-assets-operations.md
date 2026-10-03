# Managing digital assets: operational semantics (AEMaaCS)

The non-obvious behaviors behind everyday Assets operations — what copy,
move, delete, publish and upload actually do. Rendition/delivery machinery
is elsewhere ([[web-optimized-image-delivery]], [[dynamic-media-scene7-mode]]);
duplicate detection in [[duplicate-asset-detection]].

---

## Upload and naming

- ZIP extraction: ≤ **15 GB per archive**, ≤ **3 ZIPs extracted
  simultaneously**; on name clashes the dialog offers create-version /
  replace / keep-both / skip.
- Illegal characters — assets: `* / : [ \ ] | # % { } ? &`; folder names
  additionally ban `" . ^ ; + &` and tab. **`subassets` is a reserved
  folder name** (compound-asset internals).
- **~1,000 direct children per folder** is the practical ceiling —
  thousands degrade listing, moves and workflows. A `/` in a folder
  *title* breaks Column View rendering (display bug, not data loss).

## Copy vs move (asymmetric on purpose)

- **Copy = complete duplicate, minus identity**: the copy gets a new asset
  ID, creation date, and **no version history** (`jcr:uuid`, `jcr:created`
  don't carry). Name clash → auto-suffix (`Square` → `Square1`).
- **Move offers "Adjust References"** — skip it and inbound links break
  silently. Move needs **replicate permission, not just write** (moving a
  *published* asset implies republish); without it the move lands in a
  pending-approval workflow.

## Delete

- Reference check first: *"One or more assets are referenced"* blocks the
  delete; **Force Delete** overrides and leaves broken links (admins can
  remove the force option via overlay). Needs actual delete permission on
  the asset — modify-only can't.

## Publish

- **Publishing an asset still in processing publishes the original only —
  renditions are missing** on Publish; republish after processing.
- **Empty folders never publish.**
- Manage Publication = now/later scheduling + optional inclusion of
  references; when unpublishing, shared references are left out so other
  published assets don't break.

## Versions

Created automatically on external-app edit + re-upload, metadata edits,
and desktop-app checkout saves; manually via Save as Version
(label/comment). A version stores **metadata + renditions** along with the
binary; revert = restore to current.

## Bulk operations go async

Folder move/copy/delete touching **> 150 assets** becomes a background job
(run now or schedule; tracked in the **Assets Jobs console**; progress is
checkpointed and recoverable). During move/delete the affected folders are
access-restricted.

## Small gotchas worth keeping

- **Expired assets are hidden in the web UI only** — desktop app / Asset
  Link still serve them unless `hideExpiredAssets=true` is configured.
- Private-folder sharing: the **membership list overrides ACL-based
  sharing**; read access ≠ ability to share.
- Video annotations don't support **MXF** (HTML5-playable formats only);
  annotation suggestions need read on `/home` for non-admins.
- Timeline consolidates workflows, comments/annotations, activity, and
  versions; in Collections it exists **only for top-level collections**.
- `sling:OrderedFolder` is not supported for Experience Cloud sharing.

## Exam checklist

- Copy drops identity + versions; move needs **replicate** permission and
  Adjust References.
- Publish-during-processing = original without renditions; empty folders
  never publish.
- > 150 assets → async job; ~1,000 children/folder guidance; `subassets`
  reserved.
- Force Delete = broken links by consent; reference check is the default.

## References
- [Manage digital assets (Adobe docs, AEMaaCS)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/assets/manage/manage-digital-assets)

Docs-based (page read Oct 2026), behaviors not re-verified on an
environment.
