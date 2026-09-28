# Show / Hide Fields in AEM Touch UI Dialogs

How to make a field in a `cq:dialog` appear or disappear in real time based on another field's value, using the OOTB `cq-dialog-dropdown-showhide` mechanism.

Use this for any dialog where one field's visibility depends on another (commonly: a select drives the visibility of other fields). It works live as the user types/selects — no save+reopen required.

---

## The two roles

| Role | What it does |
|---|---|
| **Trigger** | The controlling field. When its value changes, it notifies its targets. Almost always a `select` (or a checkbox, with `cq-dialog-checkbox-showhide`). |
| **Target** | A field or container whose visibility depends on the trigger's current value. Multiple targets can listen to one trigger. |

## Required attributes

### Trigger
- `granite:class="cq-dialog-dropdown-showhide"` — marks the field as a showhide trigger. If the field already has classes (e.g. `cmp-form-text__types`), include them too: `granite:class="cmp-form-text__types cq-dialog-dropdown-showhide"`.
- Child `<granite:data>` node with `cq-dialog-dropdown-showhide-target=".your-target-class"` — a CSS selector telling the framework which targets to look at. Must start with `.` because it's matched as a CSS class selector.

### Target
- `class="hide your-target-class"` — the target class (matching the selector on the trigger) plus `hide` so the field starts hidden until the framework decides otherwise.
- `showhidetargetvalue="<value>"` — the trigger value that causes this target to be shown. If the trigger's current value matches this, the `hide` class is removed.

### Which names are fixed vs. arbitrary

AEM's framework JS at `/libs/cq/gui/components/authoring/dialog/dropdownshowhide/clientlibs/...` is hardcoded to look for specific names. Rename any of the fixed ones and the framework will not fire.

| Name | Fixed or arbitrary? | Notes |
|---|---|---|
| `cq-dialog-dropdown-showhide` (class on trigger) | **Fixed** | Use `cq-dialog-checkbox-showhide` instead for checkbox-driven showhide. |
| `cq-dialog-dropdown-showhide-target` (data attr on trigger) | **Fixed** | Paired with the class above. |
| The target class itself, e.g. `.cmp-form-text__date-showhide-target` | **Arbitrary** | You choose it. Convention: prefix with your component namespace and suffix with `-showhide-target` so it doesn't collide across dialogs. |
| `showhidetargetvalue` (attribute on target) | **Fixed** | The attribute name; its value is your choice (must match a trigger value). |
| `hide` (class on target) | **Fixed** | The framework toggles this class on/off. |

> The target attributes (`class`, `showhidetargetvalue`) must be passed through to the rendered HTML. **Use `sling:resourceType="granite/ui/components/foundation/container"`** as the wrapper, not the `coral/foundation/container` variant — the latter filters unknown attributes and your `showhidetargetvalue` will silently be dropped, leaving the field hidden forever.

---

## Minimal example

Trigger select + one target field:

```xml
<type
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/select"
    granite:class="cq-dialog-dropdown-showhide"
    fieldLabel="Type"
    name="./type">
    <granite:data
        jcr:primaryType="nt:unstructured"
        cq-dialog-dropdown-showhide-target=".my-date-showhide-target"/>
    <items jcr:primaryType="nt:unstructured">
        <text  jcr:primaryType="nt:unstructured" text="Text" value="text"/>
        <date  jcr:primaryType="nt:unstructured" text="Date" value="date"/>
    </items>
</type>

<dateOnlyField
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/datepicker"
    class="hide my-date-showhide-target"
    showhidetargetvalue="date"
    fieldLabel="Date value"
    name="./dateValue"/>
```

Result: `dateOnlyField` is only visible when `type === "date"`.

---

## Grouping multiple targets

To show/hide several fields together, wrap them in a container that is itself the target:

```xml
<dateConstraintsGroup
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/foundation/container"
    class="hide my-date-showhide-target"
    showhidetargetvalue="date">
    <items jcr:primaryType="nt:unstructured">
        <fieldA .../>
        <fieldB .../>
        <fieldC .../>
    </items>
</dateConstraintsGroup>
```

All three fields appear/disappear together based on the trigger's value.

---

## Multiple values from one trigger

Different targets can react to different values of the same trigger by giving each its own `showhidetargetvalue`:

```xml
<staticGroup    class="hide title-style-showhide-target" showhidetargetvalue="static"  .../>
<dynamicGroup   class="hide title-style-showhide-target" showhidetargetvalue="dynamic" .../>
```

When the trigger is `static`, only `staticGroup` is shown; switch to `dynamic` and the other appears.

See `ui.apps/.../components/executivehealth/multistepform/v1/multistepform/_cq_dialog/.content.xml` for a working in-repo example.

---

## Nesting (AND logic between conditions)

A target field can only have **one** `showhidetargetvalue`. If you need a field to be visible only when *condition A AND condition B* are both true, nest containers:

```xml
<outerGroup
    class="hide outer-target"
    showhidetargetvalue="date">         <!-- driven by ./type -->
    <items>
        <maxDateType
            granite:class="cq-dialog-dropdown-showhide"
            name="./maxDateType" ...>
            <granite:data cq-dialog-dropdown-showhide-target=".inner-target"/>
            <items>...</items>
        </maxDateType>
        <innerGroup
            class="hide inner-target"
            showhidetargetvalue="specific">   <!-- driven by ./maxDateType -->
            <items>
                <maxDate .../>
            </items>
        </innerGroup>
    </items>
</outerGroup>
```

`maxDate` is visible only when `type === "date"` AND `maxDateType === "specific"`.

---

## Driving from a field in a different tab / parent dialog

The trigger and target do **not** need to live in the same tab. Sling resource merging lets you override a single field deep inside a parent dialog without copying the rest of the tree.

If the parent has the trigger field at e.g. `cq:dialog/content/items/tabs/items/properties/items/columns/items/column/items/fieldType`, the override looks like this:

```xml
<properties jcr:primaryType="nt:unstructured">
    <items jcr:primaryType="nt:unstructured">
        <columns jcr:primaryType="nt:unstructured">
            <items jcr:primaryType="nt:unstructured">
                <column jcr:primaryType="nt:unstructured">
                    <items jcr:primaryType="nt:unstructured">
                        <fieldType
                            jcr:primaryType="nt:unstructured"
                            granite:class="cmp-form-text__types cq-dialog-dropdown-showhide">
                            <granite:data
                                jcr:primaryType="nt:unstructured"
                                cq-dialog-dropdown-showhide-target=".my-target"/>
                        </fieldType>
                    </items>
                </column>
            </items>
        </columns>
    </items>
</properties>
```

Only the attributes/children listed are touched. `sling:resourceType`, `name`, `fieldLabel`, the `<items>` (select options), etc. are inherited from the parent. **Keep any existing `granite:class` value when adding `cq-dialog-dropdown-showhide`** — attributes overwrite, they don't merge.

---

## Checklist when adding show/hide

1. Trigger field: `granite:class` contains `cq-dialog-dropdown-showhide` (plus any pre-existing classes).
2. Trigger field has a `<granite:data>` child with `cq-dialog-dropdown-showhide-target=".some-class"`.
3. Target wrapper uses `sling:resourceType="granite/ui/components/foundation/container"` (not the `coral/` variant) so custom attributes pass through.
4. Target has both `class="hide some-class"` and `showhidetargetvalue="<value>"`.
5. Removed any old server-side `granite:rendercondition` that was solving the same problem — leaving both can hide the field at render time and make the live JS irrelevant.

---

## Why not `granite:rendercondition`

`granite/ui/components/coral/foundation/renderconditions/property` is **server-side**: it's only evaluated when the dialog is rendered. The field decides whether to render itself based on the saved JCR property value. Changing the controlling field in the dialog and expecting the dependent field to appear/disappear without saving will not work — the user has to save and reopen the dialog. Use rendercondition only when the visibility truly doesn't change during a single dialog session.

---

## Variants

- **Checkbox trigger:** use `cq-dialog-checkbox-showhide` on the checkbox (instead of `-dropdown-`) and `cq-dialog-checkbox-showhide-target` on the granite:data. Targets work the same way; `showhidetargetvalue` is typically `"true"` or `"false"`.
- **Older syntax:** some examples in this repo use `class="cq-dialog-dropdown-showhide"` and `cq-dialog-dropdown-showhide-target` as a direct attribute on the trigger (no `<granite:data>` child). Both forms work; prefer the modern `granite:class` + `<granite:data>` pattern.

## What AEM ships OOTB — and what it doesn't

Only **two** trigger types ship in the OOTB framework under `/libs/cq/gui/components/authoring/dialog/`:

| Trigger class | Drives off |
|---|---|
| `cq-dialog-dropdown-showhide` | A `<select>` value |
| `cq-dialog-checkbox-showhide` | A checkbox's checked state (`"true"` / `"false"`) |

There is **no** OOTB `cq-dialog-radio-showhide`, `cq-dialog-textfield-showhide`, `cq-dialog-multifield-showhide`, etc. If your trigger is a radio group, a text input, a numberfield, a multifield, a slider, or any other field type, you have two choices:

1. **Custom dialog clientlib** — write a small JS file that listens for the trigger's `change` event and toggles a `hide` class on the target wrappers manually. See `ui.apps/.../components/content/clinicaltrials/v1/clinicaltrials/clientlibs/dialog/js/listener.js` in this repo for the in-house pattern: a `cq:ClientLibraryFolder` registered in a category and pulled in via `extraClientlibs` on the dialog.
2. **ACS Commons `dependsOn`** — if the project includes ACS Commons, the `dependsOn` plugin generalizes the pattern with expressions like `dependsOn="@type === 'date'"` and `dependsOnAction="visibility"`. Works on any field type with no custom JS.

Reach for the OOTB classes whenever the trigger is a select or checkbox; only drop to a clientlib or `dependsOn` when the trigger type forces it.