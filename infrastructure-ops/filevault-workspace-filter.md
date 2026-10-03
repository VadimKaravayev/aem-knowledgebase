# FileVault workspace filter (filter.xml)

The authoritative semantics of `META-INF/vault/filter.xml` from the
Jackrabbit FileVault docs — what Package Manager's filter UI
([[package-manager]]) and the Maven packaging rules
([[aemaacs-content-package-structure]], [[oak-indexing-aemaacs]]) compile
down to.

---

## Shape

`<workspaceFilter version="1.0">` (XSD: `…/filevault/xsd/workspacefilter-1.0.xsd`)
containing `<filter>` elements. Per filter:

- **`root`** (mandatory) — absolute JCR path, the covered subtree.
- **`mode`** (optional, lowercase) — import behavior:
  - **`replace`** (default): covered content is fully replaced — *including
    deletions* (see the matrix),
  - `merge_properties`: only **new** content added, existing untouched,
  - `update_properties`: new added, existing **updated**, nothing deleted,
  - `merge`, `update`: **deprecated** aliases of the above two — most blog
    advice still names these.
- **`type="cleanup"`** — the rule is ignored by package-type
  auto-detection and orphaned-filter validation (for maintenance filters
  that shouldn't count as "this package owns that path").

## Include/exclude rules

- `pattern` = **regex against the full JCR path** (not a glob): absolute
  (starts `/`) or starting with a wildcard (relative — and a fully
  relative pattern forces traversal of *all* children on export).
- `matchProperties="true"` (FileVault ≥ 3.1.28) matches **property** paths
  instead of node paths — property-level filtering, e.g. keep an
  environment-tuned property while replacing its node's other properties.
- **Evaluation: sequential, the LAST matching element wins** — same
  last-match-wins shape as dispatcher `/filter`. **The first rule's type
  sets the default**: first rule include → unmatched paths are excluded,
  first rule exclude → unmatched paths are included. (Package Manager's
  "include rules are exclusive" behavior is this rule in disguise.)

## The install state matrix (the part that deletes content)

| Path covered by filter | In package | In repo | Result |
|---|---|---|---|
| yes | yes | yes | **overwritten** |
| yes | **no** | yes | **REMOVED from the repository** |
| yes | yes | no | created |
| no | yes | — | **not touched** (even though it's in the package!) |

Row 2 is the "installing the package deleted my content" incident: with
`replace` mode, coverage without content is a deletion instruction. Row 4
is its quiet sibling: content *in* the package but outside the filters
silently doesn't install.

## Export vs import

The same filter drives both: on **export** it decides what gets serialized
(only matching nodes are traversed); on **import** it decides what gets
deserialized *and* what gets removed (matrix above). A filter edited for a
convenient export therefore changes what the install owns.

## Uncovered ancestors

When intermediate nodes must be created on install: the node type from the
package's `.content.xml` if provided, else the parent's default child type
or `nt:folder` (≥ 3.4.4), and existing ancestors are left untouched.

## Exam checklist

- `mode`: replace (default, deletes) / merge_properties (add-only) /
  update_properties (add+update, no delete); `merge`/`update` deprecated.
- Patterns are **regex on full paths**; last match wins; first rule's type
  sets the default.
- **Covered + absent from package = removed** on install; in package but
  uncovered = never installed.
- `matchProperties` (≥ 3.1.28) filters at property level.
- `type="cleanup"` excludes a rule from package-type detection/validation.

## References
- [Workspace Filter (Apache Jackrabbit FileVault docs)](https://jackrabbit.apache.org/filevault/filter.html)

Docs-based (page read Oct 2026).
