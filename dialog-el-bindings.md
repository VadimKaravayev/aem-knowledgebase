# EL Bindings Available in a Granite UI Authoring Dialog

What implicit objects you can reference inside `${...}` expressions in a Touch UI (`cq:dialog`) field — the binding behind `granite:hide="${cqDesign.descriptionHidden}"` and friends.

Use this when authoring `granite:hide`, `granite:rendercondition` (`.../renderconditions/simple`), or any `value="${...}"` / attribute expression in a dialog field, and you need to know what's in scope and what each object actually points at.

---

## Where these expressions run

Expressions in dialog field `.content.xml` are **evaluated server-side** when Granite UI builds the dialog markup — not in the browser. The evaluator is `com.adobe.granite.ui.components.ExpressionResolver` (impl `ExpressionResolverImpl`), a `javax.el` engine wired with a fixed chain of `ELResolver`s for the base implicit objects, plus a `CustomVariableELResolver` for variables that rendering scripts inject at request time (this is how `cqDesign` gets in).

This is a different mechanism from `cq-dialog-dropdown-showhide` (see [dialog-showhide-fields.md](dialog-showhide-fields.md)), which is **client-side JS** reacting to live field changes. EL bindings are resolved **once, at dialog render time**, and cannot react to the author changing another field. Rule of thumb:

- **Visibility depends on the component's policy / a static request fact** → EL + `granite:hide` (server-side, this doc).
- **Visibility depends on another field's live value** → `cq-dialog-dropdown-showhide` (client-side, the other doc).

## The base implicit objects (verified — built into `granite.ui.commons`)

These are the **fixed** implicit objects, each provided by a dedicated `ELResolver` in `com.adobe.granite.ui.commons` (verified against bundle **5.10.44**; see *How this was verified* below). They are present in **every** Granite UI EL expression, regardless of component or policy.

| Object | Provided by | What it resolves to |
|---|---|---|
| **`requestPathInfo`** | `RequestPathInfoELResolver` | Sling `RequestPathInfo` of the current request — `.selectorString`, `.suffix`, `.extension`, `.resourcePath`. The supported way to reach the edited content path (via `.suffix`). |
| **`param`** / **`paramValues`** | `ExternalImplicitObjectELResolver` | Request parameters (single / multi-value). `param.item` carries the `?item=` content path. |
| **`header`** / **`headerValues`** | `ExternalImplicitObjectELResolver` | Request headers (single / multi-value). |
| **`cookie`** | `ExternalImplicitObjectELResolver` | Request cookies. |
| **`sling`** | `ExternalImplicitObjectELResolver` | Exposes `sling.request` / `sling.response`. |
| **`querystring`** | `QueryStringELResolver` | The current request's query string. |
| **`state`** | `StateELResolver` | The client state of the current request. |
| **`tenant`** | `TenantELResolver` | The tenant of the current request. |
| **`userPrefs`** | `UserPreferencesELResolver` | The current user's preferences. |
| **`empty`** | EL operator (not a binding) | Standard EL — `${empty cqDesign.allowedTypes}`. Always available. |

That is the **complete** set of statically-registered objects. Notably **absent**: `resource`, `properties`, `currentPage`, `currentStyle`, `request`, `resourceResolver`, `xssAPI`. Those are HTL/JSP script bindings — they are **not** part of the Granite UI dialog EL context. Don't reach for them in a `granite:hide`.

### `granite:` is a function namespace, not an object

There is **no** `granite` implicit *object* in dialog EL (so `${granite.csrfToken}` does **not** work — that's HTL thinking). What exists is a set of EL **functions** under the `granite:` prefix, provided by `ExpressionResolverImpl`:

`granite:url`, `granite:absUritemplate`, `granite:uritemplate`, `granite:encodeURIComponent`, `granite:encodeURIPath`, `granite:relativeParent`, `granite:concat`, `granite:contains`, `granite:containsIgnoreCase`, `granite:toJSONArray`, `granite:toQueryString`

Used as `${granite:url(resource.path)}` — function-call syntax, not property access.

## `cqDesign` and other custom variables — injected per render, not fixed bindings

`cqDesign` is **not** a built-in resolver. It is a **custom variable injected at render time** through this mechanism:

- `com.adobe.granite.ui.components.ExpressionCustomizer` — request-scoped helper. `ExpressionCustomizer.from(request).setVariable(name, value)` registers a named variable for the duration of the request.
- `CustomVariableELResolver` — resolves *any* name by looking it up in the request's `ExpressionCustomizer`. It carries no hardcoded names; it's the open door through which arbitrary variables enter EL.

### Where `cqDesign` is actually set (traced)

`cqDesign` is a literal in **`com.day.cq.wcm.api.WCMFilteringResourceWrapper`** (bundle `com.day.cq.wcm.cq-wcm-api`, verified by decompiling the class). The chain:

1. `/libs/cq/gui/components/authoring/dialog/dialog.jsp` includes `…/properties/FilteringResourceWrapper.jsp`, which wraps the dialog resource in `WCMFilteringResourceWrapper`.
2. The wrapper's `registerCustomVariables(resource, request)` resolves the edited component's design via `Designer.getStyle(resource)` → `getDesignProperties(...)` (a `ValueMap`).
3. It calls `ExpressionCustomizer.from(request).setVariable("cqDesign", designProperties)`.
4. At EL eval, `CustomVariableELResolver` resolves `${cqDesign.*}` against that.

So `cqDesign` is the **design/policy `ValueMap`** of the component being edited — reliably present in a component edit dialog, empty `ValueMap` if no policy applies. The Core Components teaser/title/image dialogs rely on exactly this.

> Earlier scans that grepped the `.jar` bytes directly missed this — JARs are ZIP-compressed, so a literal like `cqDesign` does not appear in the raw file. You must decompress class entries first (`unzip -p jar '*.class' | grep`, `zipgrep`, or extract + `javap`). A raw `grep -a` over `.jar` files gives false negatives.

### The "friends" — every custom variable injected platform-wide

Found by decompressing all Felix-cache bundles and extracting the `String` arg of every `ExpressionCustomizer.setVariable(...)` call (verified, AEMaaCS SDK):

| Variable | Injected by (bundle) | Value / when present |
|---|---|---|
| **`cqDesign`** | `WCMFilteringResourceWrapper` (`com.day.cq.wcm.cq-wcm-api`) | Design/policy `ValueMap` of the edited component. Present in component edit dialogs. |
| **`scaffold`** | a precompiled script (`aem-precompiled-scripts`) | Scaffolding edit context. Present only in scaffold editing. |
| **`imageDelegate`** | Core Components (`com.adobe.cq.core.wcm.components.core`) | Image-component delegate. Present in the Core Image v3 dialog. |

That's the complete set of *Java/precompiled-script* injectors on a stock SDK. Project code (or an uncompiled `/libs`/`/apps` JSP/HTL script) can add more by calling `setVariable` itself — which is why the set is **open-ended by design**, and why you enumerate it by scanning rather than trusting a fixed list.

## Gotcha: there is no `properties` / `resource` shortcut to authored content

Because `resource`/`properties` are **not** in the dialog EL context at all (verified above), you cannot read the edited component's saved values with `${properties.jcr:title}` from a dialog field. To act on the saved content you must resolve it yourself — take the content path from `${requestPathInfo.suffix}` (or `${param.item}`) in a custom `datasource` / rendercondition and load that resource in Java. To gate on the **policy**, use `cqDesign`, which is exactly why Core Components drives field visibility off `cqDesign.*`.

## Worked example (Core Components teaser v2)

`content/.../teaser/v2/teaser/_cq_dialog/.content.xml` gates fields entirely off the policy:

```xml
<descriptionGroup granite:hide="${cqDesign.descriptionHidden}" .../>
<title             granite:hide="${cqDesign.titleHidden}"       .../>
<pretitle          granite:hide="${cqDesign.pretitleHidden}"    .../>
<actions           granite:hide="${cqDesign.actionsDisabled}"   .../>
<!-- compound conditions with empty / or / not: -->
<titleType granite:hide="${!cqDesign.showTitleType or (empty cqDesign.allowedHeadingElements and empty cqDesign.allowedTypes)}" .../>
```

The matching booleans (`descriptionHidden`, `titleHidden`, …) are written by the component's **design dialog** (`_cq_design_dialog`) into the policy. So a template author ticks "Hide Description" once, and every content author on that template loses the Description field. `granite:hide` removes the field **server-side** — it isn't rendered at all, not merely CSS-hidden.

## Forcing a field on/off regardless of policy

When proxying / supertyping a Core Component and you want a field always hidden (independent of any policy), you have two levers:

1. **Policy layer** — set the relevant `cqDesign` property (`descriptionHidden=true`, …) in your component's policy. The Adobe-blessed way; no dialog overlay needed.
2. **Dialog overlay** — override the field in your component's `_cq_dialog` with a static `granite:hide="true"`, or replace it with a hidden Granite field. Use when there's no policy property for what you want, or you need it forced regardless of how the template is configured.

## Why the list is open-ended (but the *base* is fixed)

The **base** implicit objects are a fixed, hardcoded set of `ELResolver`s compiled into `granite.ui.commons` (the table above — verified, not guessed). What is **open-ended** is the *custom-variable* layer: any rendering script or wrapper can call `ExpressionCustomizer.from(request).setVariable(name, value)`, and `CustomVariableELResolver` will resolve that name in EL. On a stock SDK the injected variables are `cqDesign` (component dialogs), `scaffold` (scaffold editing), and `imageDelegate` (Core Image dialog) — each present only in its own context.

So: trust the base table everywhere; treat any **other** name (`cqDesign`, `scaffold`, `imageDelegate`, or anything project code injects) as context-specific — present only because some wrapper/script in that exact context called `setVariable`.

## How this was verified

Against a local AEMaaCS SDK (Author), Granite UI commons **5.10.44**:

1. `GET /system/console/bundles/<symbolic-name>.json` → bundle id + `Bundle Location` (e.g. `launchpad:resources/install/.../com.adobe.granite.ui.commons-5.10.44.jar`).
2. Located the active JAR in the Felix cache: `crx-quickstart/launchpad/felix/bundle<id>/version0.0/bundle.jar`, unzipped it.
3. **Base implicit objects** — `javap -p -c` on each `com/adobe/granite/ui/components/impl/el/*ELResolver.class` in `granite.ui.commons` → the implicit-object name each one registers (`requestPathInfo`, `querystring`, `state`, `tenant`, `userPrefs`, and `param`/`header`/`cookie`/`sling` from `ExternalImplicitObjectELResolver`); and on `ExpressionResolverImpl.class` → the `granite:*` function names.
4. **Custom variables (`cqDesign` & friends)** — found the injectors by decompressing class entries (NOT raw `grep` — see below) across the whole Felix cache: `for d in bundle*/version*/bundle.jar; do unzip -p "$d" '*.class' | grep -qa ExpressionCustomizer && echo "$d"; done`. Then for each hit, `javap -c` the referencing class and read the `// String …` ldc immediately before each `ExpressionCustomizer.setVariable`. Yield: `cqDesign` (`com.day.cq.wcm.cq-wcm-api` / `WCMFilteringResourceWrapper`), `scaffold` (`aem-precompiled-scripts`), `imageDelegate` (`com.adobe.cq.core.wcm.components.core`).
5. **`/libs` script source** (e.g. `dialog.jsp`, `global.jsp`) read via DavEx: `GET /crx/server/crx.default/jcr:root/<path>/jcr:content/jcr:data` (a script URL executes; the `jcr:data` child returns source). This traced `dialog.jsp` → `FilteringResourceWrapper.jsp` → the wrapper class.

### ⚠ The grep gotcha that produced a wrong answer first time

A first pass used `grep -a "cqDesign" bundle.jar` directly on the JARs and got **zero hits** — leading to the false conclusion "`cqDesign` is in no bundle." **JARs are ZIP/DEFLATE-compressed**, so class-constant strings are not present in the raw bytes. Always decompress first: `unzip -p jar '*.class' | grep`, `zipgrep pattern jar`, or extract + `javap`. (`zipgrep` is correct but slow — one `unzip` per entry; the `unzip -p '*.class'` one-shot per jar is much faster across hundreds of bundles.)

To re-verify on a different SDK build, repeat steps 1–4 (bundle versions differ; the resolver/wrapper class names have been stable across 6.5 / AEMaaCS).

## Related

- [dialog-showhide-fields.md](dialog-showhide-fields.md) — the client-side, live-reacting alternative for field visibility.
- [xml-escaping-in-vault-content.md](xml-escaping-in-vault-content.md) — escaping `${`/special chars inside `.content.xml` attribute values.