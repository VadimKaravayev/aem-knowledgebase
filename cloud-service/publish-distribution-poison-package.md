# One malformed XML file in a content package makes the Install mutable content step loop

A content package can install fine on **author** and still be rejected on **publish** when it is
distributed. Each attempt fails for a few minutes, then the item is dropped and marked "Successfully
applied" even though nothing was imported. The deploy then sends the whole batch of mutable content
packages again. The loop runs for hours, and Deploy to Dev fails at its **Install mutable content** step;
**every deploy that ships the broken package fails the same way**. Because each item only stays in
the queue for minutes, **you will not find a stuck item** in Tools → Deployment → Distribution.

Related: queue mechanics in [[content-distribution-journal]], the `/var` variant of the same symptom
in [[aemaacs-build-deploy-debugging]], what installs when in [[rolling-deployment-two-version-overlap]].

**Sourcing:** observed on a real AEMaaCS dev environment (program p153802, 2026-10-09, AEM
2026.8). Diagnosed from the author `aemerror` log and confirmed by unpacking the package. The empty
queue was checked in the Distribution UI the next day. Inferred, not shown in the log: that the
re-sends come from the Install mutable content step retrying, and that the loop is why that step fails.

---

## Signature

In the **author** `aemerror` log, starting a few minutes after the deploy that added the package:

```
*WARN* ... org.apache.sling.distribution.journal.impl.publisher.DistributionPublisher [publish-internal]
Message: Failed attempt (N/infinite) to import the distribution package PackageMessage(... reqType=ADD,
pkgType=journal_filevault, ... paths=[/etc/packages/<group>/<name>-<version>-cp2fm-converted.zip] ...)
because of 'Error trying to extract package ... DocViewParser$XmlParseException: <parser message>'
```

- `(N/infinite)` is misleading. Each package ID got about 14 attempts over 5–10 minutes, then
  `Successfully applied package with id <same id>, type ADD, paths [<same path>]`, which means it was
  dropped, not imported. About 20 s later the same path goes out again under a **new** package ID.
  Seen: 715 failed attempts over ~43 package IDs between 11:07 and 23:58, all one path.
- The **other mutable content packages loop too**, and succeed every time (seen: the Phrase and
  Translated `ui.content` re-distributed about 45 and 30 times). The deploy retries the whole batch.
  Packages whose version didn't change (e.g. WKND 4.0.4 `ui.content`) are skipped and not re-sent.
- The broken package never reached `preview-internal`. The batch stops before preview.
- `-cp2fm-converted.zip`: the package Cloud Manager's converter produced from your mutable content
  package. Its name tells you which source package is broken.
- The pipeline deploy shows progress (new author pods roll out, their bundles start), then sits in
  Deploy to Dev: the UI shows "Updating indexes" for about 2 hours, then the step that fails is
  **Install mutable content**. Don't chase indexes. **Two consecutive deploys failed the same way.**

Quick triage:

```bash
grep -c 'Failed attempt' author_aemerror.log
grep 'Failed attempt' author_aemerror.log | grep -oE 'paths=\[[^]]*\]' | sort | uniq -c
grep -m1 'Failed attempt' author_aemerror.log | grep -oE "because of '.{0,300}"
# loop vs stuck: many distinct ids = loop
grep 'Failed attempt' author_aemerror.log | grep -oE 'pkgId=[^,]+' | sort -u | wc -l
```

## Cause seen: whitespace before the XML declaration

`DocViewParser$XmlParseException: The processing instruction target matching "[xX][mM][lL]" is not allowed.`
means a `.content.xml` has **something before `<?xml ...?>`**: a leading newline, a BOM, or a comment.
The XML spec allows the declaration only as the very first bytes.

The case seen: Adobe's WKND SPA React 1.0.3 `ui.content`,
`conf/wknd-spa-react/settings/wcm/template-types/spa-page/initial/.content.xml`, starts with `\n<?xml`.
Author installed it (pages present under `/content/wknd-spa-react`), but the publish importer rejected it.

Find every such file in an unpacked package:

```bash
find jcr_root -name '*.xml' | while read -r f; do
  [ "$(head -c 5 "$f")" != "<?xml" ] && grep -q '<?xml' "$f" && echo "$f"
done
```

Run this (or `xmllint --noout` on every `.content.xml`) on any vendored content package before you
ship it. FileVault validation in your own build doesn't catch this when `skipSubPackageValidation` is
on, which is the usual setting for vendor embeds.

## Fix, in this order

1. **Fix the XML** and re-zip the package (`META-INF` + `jcr_root`).
2. **Change the package version inside the package** (`META-INF/vault/properties.xml` `version`)
   so it is distributed as a new package rather than matched to the one already recorded. Change
   the Maven version of the vendored artifact to match, with a suffix that marks it as modified
   (e.g. `1.0.3-xmlfix`). If another package declares a dependency on this one, update it too.
3. Redeploy. There's no queue item to clear: the dropped items leave nothing behind (checked in the
   Distribution UI the day after: empty).
4. After the deploy, grep the new author `aemerror` for the **old** package path. If attempts with
   the old path continue when no pipeline is running, the loop is outliving the deploy: open an
   Adobe support case with the log excerpt.

## Checklist

- Author OK + publish missing content + `Failed attempt (N/infinite)` on `publish-internal`, with
  **many package IDs for one path**, each ending in "Successfully applied" + Deploy fails at **Install mutable content** after ~2 h
  fails: one bad package is making Install mutable content loop.
- The parser message names the defect. `[xX][mM][lL]` = bytes before `<?xml`.
- The queue looks empty: items are dropped after a few minutes. "Successfully applied" after
  failures means dropped, not imported.
- Vendored content packages need their own XML check. Vendor-embed builds usually skip validation.
