# Package Manager (AEM 6.5)

The semantics behind Package Manager — snapshots, filters, AC handling,
validation — as opposed to its HTTP endpoints
([[aem-curl-operations]]). Cloud-side constraints (one-offs, 10-min
timeout, `cp2fm` frozen packages) in
[[rolling-deployment-two-version-overlap]]; build-time package anatomy in
[[aemaacs-content-package-structure]].

---

## What a package is — and isn't

A zip of repository content in **FileVault (vault) serialization**
(`jcr_root` + `META-INF` with filters, import config, properties). A build
captures **the current state only — no version history travels with a
package**, so "restore the old content from last month's package" only
works if the package was *built* last month.

## Snapshots: the part everyone half-knows

- **Install** first builds an automatic **snapshot package of whatever the
  install is about to overwrite**; **Uninstall = reinstall that snapshot**
  — it reverts to pre-install state, it does not simply "remove" content.
- **Extract Only** installs **without creating the snapshot → uninstall is
  impossible** for that package. (This is also the semantic difference
  behind cloud one-offs.)
- **Delete** removes the package *entity* from `/etc/packages` only —
  installed content stays. Delete ≠ uninstall, a classic operational
  mistake in both directions.

## Filters

Per filter: a root path + zero or more rules. **No rules = the entire
subtree.** **Include rules are exclusive**: once any include exists, *only*
regex-matching paths under the root are packaged; everything else under
that root is out. Exclude rules subtract matches. **Coverage** previews
what the filters actually catch — use it before build, not after install.

## AC handling (five modes)

`Ignore` (**the default**; leave repo ACLs alone — package ACEs not applied) ·
`Overwrite` (replace the node's ACLs with the package's) ·
`Merge` (combine; existing principals keep their entries) ·
`MergePreserve` (merge, adding only entries for principals not yet present)
· `Clear` (strip ACLs). Set at package level (Advanced) and overridable at
install time. Typical project packaging uses `merge`
([[aemaacs-content-package-structure]]).

## Install options and friends

- Install dialog: AC handling override, **Save Threshold** (transient
  nodes before intermediate saves — tune for huge packages), **Extract
  Subpackages**, dependency handling, Extract Only.
- **Test Install = dry run** — simulates and reports to the activity log,
  changes nothing.
- **Validate** before install runs three validators: **OSGi package
  imports** (will the bundles resolve), **overlays** (does the package
  clobber an existing `/apps` overlay), **ACLs** (what permissions change).
- **Build** overwrites the package zip from current repo state; **Rewrap**
  refreshes metadata (description, thumbnail) *without* touching content —
  why a rewrapped package can be old content with a new description.
- **Replicate** pushes the package to publish (installs there via the
  replication receiver).

## Permissions

Creating/modifying/installing needs **full rights minus delete on
`/etc/packages`** plus rights on the covered content nodes. Adobe's
caution: package rights ≈ read *and overwrite* anything the filters can
reach — grant per dedicated subtree, or it's an ACL bypass by zip.

## Operational extras

- Filesystem install: drop a zip into `crx-quickstart/install/` — installed
  immediately on a running instance, or **at startup in alphabetical
  order** (name packages to order them).
- **Bulk DAM content**: disable the asset workflow launcher
  (`com.day.cq.workflow.launcher.impl.WorkflowLauncherImpl` in the OSGi
  console) during install, re-enable after — otherwise every imported
  asset fires DAM Update Asset and the instance grinds.
- UIs: Tools → Deployment → Packages, CRXDE Lite switcher, or
  `/crx/packmgr/` direct.

## Exam checklist

- Uninstall restores the **auto-snapshot** (pre-install state); Extract
  Only = no snapshot = no uninstall; Delete removes the package, never the
  content.
- Packages carry **no version history** — only state at build time.
- Include rules are exclusive; no rules = whole subtree; Coverage to
  verify.
- AC handling: Ignore/Overwrite/Merge/MergePreserve/Clear.
- Validate = OSGi imports + overlays + ACLs; Test Install = dry run.
- `/etc/packages` full-minus-delete + content rights; scope by subtree.
- `crx-quickstart/install/` installs alphabetically at startup; disable
  the workflow launcher for bulk DAM imports.

Version note: the **6.4 page is substantively identical** (same five AC
modes with Ignore default, same snapshot/validate/install-folder behavior);
its one era marker is the **Package Share → Software Distribution**
replacement — package downloads come from Software Distribution, Package
Share is dead.

## References
- [Package Manager (Adobe docs, AEM 6.5)](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/sites/administering/contentmanagement/package-manager)
- [Package Manager (Adobe docs, AEM 6.4 — same content, confirms defaults)](https://experienceleague.adobe.com/en/docs/experience-manager-64/administering/contentmanagement/package-manager)

Docs-based (page read Oct 2026), behaviors not re-verified on an instance.
