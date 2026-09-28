# Granite `foundation-picker-control`: opening a stock content picker from custom console UI

When a custom console page or wizard needs "pick pages / assets / content fragments", do **not**
build a picker. Any clickable element can open one of Granite's stock pickers and route the
selection to your JS through the `foundation-picker-control` contract. Everything visual (tree
columns, search, multi-select) comes from the stock picker; your code supplies only a trigger
element and a named handler.

## The contract: three attributes + one registry entry

Trigger element (any button; a `coral-buttonlist-item` inside a menu popover works):

```html
<button is="coral-buttonlist-item" icon="documentFragment"
        class="foundation-picker-control"
        data-foundation-picker-control-action='{"name":"myapp.addcfs","data":{}}'
        data-foundation-picker-control-src="/apps/myapp/content/cfpicker/picker.html?root=%2fcontent%2fdam&amp;selectionCount=multiple">
    Content fragments</button>
```

- `class="foundation-picker-control"` — Granite's `foundation.js` attaches the picker behavior
  to the element's click.
- `data-foundation-picker-control-src` — URL of the picker UI; Granite fetches it and shows it
  in a modal overlay. Address **stock** picker definitions through `/mnt/overlay/...` (so
  customer overlays apply); a definition you ship yourself is addressed by its own path.
  URL-encode the query values.
- `data-foundation-picker-control-action` — JSON with the **name** of the handler to call on
  confirm. A name, not a function: it is resolved in the foundation registry at submit time,
  which decouples markup from JS.

Handler registration (must run before the first pick — register on `foundation-contentloaded`
init, via jQuery's event bus, see the jQuery-vs-native gotcha below):

```js
$(window).adaptTo("foundation-registry").register("foundation.picker.control.action", {
    name: "myapp.addcfs",
    handler: function (name, el, config, selections) {
        var paths = (selections || []).map(function (s) { return s.value; });
        // config === the "data" object from the action attribute
    }
});
```

Flow: click → Granite fetches `src` into a modal → author selects and confirms → Granite tears
the modal down and resolves the action name in the registry → `handler(name, el, config,
selections)`; each selection carries the picked path in `.value`.

## Which stock picker to point `src` at

| Content | Picker `src` | Key params |
|---|---|---|
| Pages (`cq:Page`) | `/mnt/overlay/cq/gui/content/coral/common/form/pagefield/picker.html` | `root`, `selectionCount=single\|multiple`, `exclude` (regex) |
| XFs (also `cq:Page`) | same pagefield picker, `root=/content/experience-fragments` | **must** pass an explicit `exclude` (e.g. `%2fnonexistent`) — see [pagefield-picker-default-exclude.md](pagefield-picker-default-exclude.md); the built-in default exclusion hides the XF tree and the picker opens empty |
| Content fragments | a **clone** of `/libs/dam/cfm/content/cfpicker/picker` in your own namespace (see next section) | `root`, `selectionCount` (via the clone) |
| DAM: any asset | `/mnt/overlay/granite/ui/content/coral/foundation/form/pathfield/picker.html` | `root`, `selectionCount`, `filter` |

Pagefield vs pathfield is not interchangeable: the **pagefield** picker lists `cq:Page` nodes
only, so it cannot see DAM content at all; the **pathfield** picker walks the resource tree and
filters by node type. The pathfield `filter` vocabulary is fixed: `folder`, `hierarchy`,
`hierarchyNotFile`, `nosystem` — for DAM, `hierarchyNotFile` shows `nt:hierarchyNode` and hides
`nt:file` renditions. **Do not use pathfield for content fragments**: `dam:Asset` is a subtype
of `nt:hierarchyNode`, so every image and document shows next to the CFs and authors will
constantly pick the wrong node (in the common CF layout — a folder holding the CF plus its
images — the folder itself looks exactly like the fragment). Verified the hard way on the
Crowdin New Job wizard.

## Content fragments: clone the CFM picker, don't overlay it

AEM ships a dedicated CF picker at `/libs/dam/cfm/content/cfpicker/picker` whose column view
lists **folders and content fragments only** (images and renditions filtered out) and whose
search form is pre-filtered to `contentfragment=true`. Two catches:

- **It is single-select**: `selectionCount: "single"` is hardcoded in three places — the picker
  root, `views/column`, and `search/views/card`.
- **Overlaying it (`/apps/dam/cfm/content/cfpicker/picker`) is a trap for vendor packages**:
  `/apps/dam` is a shared namespace. If two packages ship the same overlay node (e.g. two
  translation connectors on one instance), the last install overwrites the first, and
  uninstalling either can delete the node the other depends on. Overlays there belong to the
  customer, not to a connector.

The clean fix: **copy the stock definition into your own namespace**
(`/apps/<app>/content/cfpicker/picker/.content.xml`) with two changes — parameterize
`selectionCount` in all three places as
`${empty param.selectionCount ? 'single' : param.selectionCount}`, and point the two view `src`
URLs at your copy's path. The CF-only behavior travels with the copy for free: it comes from the
referenced `/libs` **components** (`dam/cfm/components/cfpicker/datasources/children`,
`.../columnitem`, `.../datasources/search`), which resolve by `sling:resourceType` from any
content path. The column `src` must append
`&selectionCount=${granite:encodeURIComponent(...)}` itself (each column render is a separate
request and the EL reads that request's params); the search card `src` already forwards
`${querystring}` wholesale.

Packaging gotchas for the clone: the intermediate folder (`content/cfpicker/`) needs its own
`.content.xml` (e.g. `sling:OrderedFolder`) — a bare directory becomes `nt:folder`, which cannot
hold the `nt:unstructured` picker node and fails filevault validation
(`jackrabbit-nodetypes`). Get the stock definition to copy from
`http://<host>/libs/dam/cfm/content/cfpicker/picker.-1.json`.

Cross-folder multi-select caveat (stock behavior, applies to every Coral columnview picker):
checked items are dropped when their column unloads, i.e. when the author navigates into a
different folder. Multi-picking across folders goes through the picker's **search** view;
column-view multi-select works within one column.

## Post-pick validation is on you

Even the CFM picker lets the author check a folder, and no pagefield `filter` means "only XF
variations" — anything pickable in the tree can come back. Validate after the pick, cheaply, with one
`<path>/_jcr_content.json` GET per selection (which also yields `jcr:title` for display):

- **Content fragment**: `jcr:content` has `contentFragment === true` (the marker CFM stamps on
  every fragment's `dam:Asset`).
- **XF variation**: `jcr:content` has `cq:xfVariantType` or `cq:xfMasterVariation`.
- Treat an unreadable `jcr:content` as invalid too — it cannot be confirmed, and downstream
  processing would fail on it anyway.

Reject invalid picks with an explanatory dialog listing the skipped paths; never silently drop.

## Gotchas

- **Dialog-after-picker timing**: the action handler fires while Granite is still tearing the
  picker modal down, and that teardown closes any `coral-dialog` already open. Opening a
  duplicate/invalid-pick dialog straight from the handler makes it flash and vanish — defer it
  (`setTimeout`, ~300 ms) so the teardown settles first.
- **jQuery event bus**: register handlers in a `foundation-contentloaded` listener bound with
  jQuery (`$(document).on(...)`). Granite triggers its lifecycle events through jQuery's
  `trigger()`, which never reaches a native `document.addEventListener` — the native listener
  fails silently.
- **Menu popovers don't close themselves**: if the trigger sits in a `coral-popover` menu, the
  same click that opens the picker leaves the popover open behind it; close it in a delegated
  click handler on the menu items.
- **`lastModified` differs by content type**: pages stamp `cq:lastModified` on `jcr:content`, a
  `dam:AssetContent` (CFs) stamps `jcr:lastModified` — read `cq:lastModified ||
  jcr:lastModified` if you record the reviewed revision.

Confirmed on the Crowdin connector New Job wizard (Pages / Experience fragments / Content
fragments menu; the CF item uses a connector-owned clone of the CFM picker at
`/apps/crowdin/content/cfpicker/picker`), July 2026. The clone-with-parameterized-selectionCount
approach matches what the translated.com connector does, minus their `/apps/dam` overlay.