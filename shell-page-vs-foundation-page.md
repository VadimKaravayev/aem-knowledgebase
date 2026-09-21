# Granite Page Bases: getting the global nav header, titlebar, and left rail

Which `sling:resourceType` to use for a **custom standalone console page** (a `cq:Page` reached by a friendly/vanity URL) when you want it to look and navigate like a native AEM admin console — global nav header, secondary titlebar, collapsible left rail — without building a full collection list.

The short version: **the global navigation header is rendered only by `granite/ui/components/shell/page` (and its subclass `…/shell/collectionpage`). `granite/ui/components/coral/foundation/page` renders a bare page with no shell chrome at all.** If users report "I can't get back to the rest of AEM from this page," you're on `foundation/page` and need `shell/page`.

---

## The three page bases

| resourceType | Renders | Use when |
|---|---|---|
| `granite/ui/components/coral/foundation/page` | `<html><head><body>` only. **No** `coral-shell`, no global header, no rail. Body comes from a `page/body` child. | A self-contained form/tool screen where you deliberately don't want AEM nav (e.g. an embedded settings form). |
| `granite/ui/components/shell/page` | `<coral-shell>` + dark global header (`coral--dark granite-shell-header`) + optional `betty-titlebar` + optional left rail + your `content`. | A custom console page that should sit inside AEM's navigation but isn't a collection list. **The default choice for a custom dashboard.** |
| `granite/ui/components/shell/collectionpage` | Everything `shell/page` does, **plus** a collection (`coral-table`/card grid) driven by a datasource, action bar, selection model, omnisearch wiring. | A list/grid of repository items with select + bulk actions. Heavier; fights you if you want fully custom body markup. |

`collectionpage` extends `shell/page`, so the header markup is identical between them. The header you see in any console (AEM logo + Omnisearch + Solutions switcher + Help + Inbox + User) is emitted by `shell/page`'s `page.jsp`, and it is **functional only because Granite wires it server-side** (the menubar items are pulled from `/mnt/overlay/granite/ui/content/shell/header/actions`, the home anchor's globalnav overlay is `…/shell/globalnav.html?consoleId=<consoleId>`). Pasting that markup as static HTML into a `foundation/page` gives you the *look* but dead menus — don't.

---

## `shell/page` content structure

From the docs embedded in `/libs/granite/ui/components/shell/page/page.jsp`:

```
- sling:resourceType = "granite/ui/components/shell/page"
- jcr:title          = "<shown in the betty-titlebar>"
- consoleId          = "<your-console-id>"          # drives globalnav highlight + rail cookie key
- omnisearchLocationPath = "/apps/granite/omnisearch/content/metadata/<x>"   # optional
+ head                                              # children iterated into <head>
    + clientlibs   (granite/ui/components/coral/foundation/includeclientlibs)
+ content          (granite/ui/components/coral/foundation/container)   # the page body
    + items
        + <your custom component>
+ actions
    + primary       # rendered in betty-titlebar-primary (right side), after the rail toggle
    + secondary     # rendered in betty-titlebar-secondary
+ rails             # presence of this node is what creates the left rail AND its toggle button
    + <panelName>   (granite/ui/components/coral/foundation/panel/railpanel)
```

Key behaviors verified against the JSP:

- **The titlebar (`betty-titlebar`) renders only if** `title`, `breadcrumbs`, or `rails` is present (`if (titleRes != null || breadcrumbs != null || rails != null)`). With none of them, you get the global header and your content but no secondary bar.
- **The left-rail toggle button** (the `railLeft` `coral-cyclebutton`, "Content Only" ⇄ panel) is emitted **only when a `rails` node exists**. No rails node → no toggle. So "I want the rail toggle to switch pages" ⇒ add a `rails` node.
- **Rail toggle renders as a flat icon button instead of a dropdown (threshold gotcha).** `shell/page`'s `page.jsp` emits the `<coral-cyclebutton>` with **no `threshold`**, so it keeps Coral's default of **3**. A `coral-cyclebutton` only shows its chevron + dropdown selectlist (`_coral-CycleSelect--extended`) when `items.length > threshold`. With the usual 2 items ("Content Only" + your one rail panel), `2 ≤ 3` ⇒ it collapses to **cycle mode**: a plain icon button, no chevron, no dropdown. `collectionpage` ships a clientlib that lowers the rail toggle's `threshold` to `1` (and renames its ids to `shell-collectionpage-rail-toggle-*`); plain `shell/page` does not, and there's no page-node config for it. Fix it from your own page clientlib after Coral upgrades the element:
  ```js
  var cb = document.querySelector('betty-titlebar coral-cyclebutton[icon="railLeft"]');
  if (cb) Coral.commons.ready(cb, function () { cb.threshold = 1; });  // → extended dropdown
  ```
  Both modes still toggle the rail (the items' `data-granite-toggleable-control-*` attrs do the work); `threshold` only controls flat-button vs dropdown presentation. Verified: working `collectionpage` `threshold=1` / `_coral-CycleSelect--extended`; fresh `shell/page` `threshold=3` / `_coral-CycleSelect`.
- **Put primary page actions** (e.g. a "New Order" button) under `actions/primary` as e.g. `granite/ui/components/coral/foundation/anchorbutton`. They land in the titlebar to the right of the rail toggle.
- Rail container id is `#shell-page-rail`; panels carry `class="shell-collectionpage-rail-panel"` + `data-shell-collectionpage-rail-panel="<name>"` (the `collectionpage-` prefix is shared, not a bug).

---

## Left-rail navigation between pages (the reusable recipe)

To get a left rail with links to sibling console pages (Dashboard, Configuration, … future pages), use the **DAM navigation panel** — same component the out-of-the-box consoles use:

```xml
<rails jcr:primaryType="nt:unstructured" active="${cookie[&quot;railActiveCookie&quot;].value}">
    <navigation
            granite:class="cq-rail-navigation"
            jcr:primaryType="nt:unstructured"
            jcr:title="Navigation"
            sling:resourceType="granite/ui/components/coral/foundation/panel/railpanel"
            active="${cookie[&quot;railActiveCookie&quot;].value}">
        <items jcr:primaryType="nt:unstructured">
            <navigationpanel
                    jcr:primaryType="nt:unstructured"
                    sling:resourceType="dam/gui/coral/components/commons/navigationpanel"
                    links-path="<absolute path to THIS node's navigationlinks, no leading slash>"
                    orientation="vertical" size="L">
                <navigationlinks jcr:primaryType="nt:unstructured">
                    <items jcr:primaryType="nt:unstructured">
                        <dashboard jcr:primaryType="nt:unstructured"
                                   jcr:title="Dashboard"
                                   href="/my/dashboard.html"
                                   navigation-url="/my/dashboard.html"
                                   selection-url-pattern="[/my/dashboard,/my/dashboard.html]"
                                   rel="cq-damadmin-navigation-link-action"/>
                        <!-- add more siblings here as the app grows -->
                    </items>
                </navigationlinks>
            </navigationpanel>
            <clientlibs jcr:primaryType="nt:unstructured"
                        sling:resourceType="granite/ui/components/coral/foundation/includeclientlibs"
                        categories="[cq.damadmin.navigation]"/>
        </items>
    </navigation>
</rails>
```

Gotchas confirmed in practice:

- **`links-path` is self-referential** — it points back at this same node's `navigationlinks` child (relative-to-root, **no leading slash**), e.g. `apps/cq/myapp/content/dashboard/jcr:content/rails/navigation/items/navigationpanel/navigationlinks`. Get it wrong and the rail renders empty.
- **The `cq.damadmin.navigation` clientlib is required** — without it the links render but don't navigate.
- **Links become `coral-tab` elements** with a `navigation-url` attribute; the clientlib's delegated click handler does the navigation. A programmatic `.click()` from the console does **not** reliably trigger it (Granite tracked-click flow) — test by actually clicking in the browser, not via injected JS.
- `selection-url-pattern` controls which entry highlights as active for the current URL.

---

## Including Coral on a `shell/page`

`shell/page` already pulls the clientlibs its own chrome needs, but include yours (and be explicit about Coral) in the `head/clientlibs` node:

```
categories="[coralui3,granite.ui.coral.foundation,granite.shared,<your.category>]"
```

Your own clientlib should also declare `dependencies="[coralui3]"` so its JS upgrades after Coral defines the custom elements. The page body component then renders **inner content only** (no `<html>/<head>/<body>`, no `<coral-shell>`) — those come from `shell/page`. Wrap your markup in a single scoping div and key your CSS off that class (not `body`), since you no longer own `<body>`.

### Coral icon sizing caveat (custom-element gotcha)

`coral-icon` elements **created dynamically in JS** (`document.createElement('coral-icon')` + `setAttribute('size','XS')`) often upgrade **without** the size class (`_coral-Icon--sizeXS`), so the inner SVG expands to fill its container (seen as 150–300px icons). Server-rendered `coral-icon` in HTL is fine. Fix: constrain `width`/`height` in CSS for the icons you build in JS. Same family: `coral-tag` applies its own narrow `max-width` + ellipsis (truncates "en-GB" → "en-"); override `max-width`/`overflow` on the tag and its `coral-tag-label`.

---

## Quick decision guide

- Need AEM global nav on a custom page? → **`shell/page`**, never `foundation/page`.
- Need a left rail and/or the rail toggle button? → add a **`rails`** node.
- Listing repository items with select + bulk actions? → **`collectionpage`**.
- Deliberately want a chrome-less embedded form? → **`foundation/page`** + `page/body`.

---

## The page's own component needs its Sling model package EXPORTED

**Symptom.** The shell, title bar, rail and any stock Granite actions render, but the one region backed by your own component is empty. The HTML carries the reason in an `<!--cq{…"exception":…}-->` comment: `SightlyException: Compilation errors … <Model> cannot be resolved to a type`. Build is green, bundle is `Active`, the class is in the jar.

**Root cause.** HTL resolves a model named in `data-sly-use` by FQCN through the dynamic class loader, which only sees **exported** packages. bnd exports a package only when it has an `@Version` `package-info.java`, so a `models` package without one is private and invisible to the HTL compiler — even though every OSGi component inside the same bundle uses it happily.

**Fix.** Add `package-info.java` with `@Version("1.0")` to the models package. Confirm with `/system/console/bundles/<bsn>.json` → `Exported Packages`.

**How to spot it next time.** A connector that strips the archetype's sample classes tends to delete their `package-info.java` too, and the first real Sling model lands in that now-private package months later. If a custom console region renders blank, `curl` the page and grep for `cannot be resolved to a type` before touching the markup. Confirmed on the Phrase connector's Configuration console, Sept 2026.

---

## Reference

- `/libs/granite/ui/components/shell/page/page.jsp` (read it on the instance — the doc comment at the top is the authoritative content-structure spec).
- Verified on AEMaaCS SDK, May 2026, building the Translated connector's redesigned orders dashboard.