# Granite wizard: Next stays disabled with every required field filled

## Symptom

A `granite/ui/components/coral/foundation/wizard` page. Every required field on the step holds
a value, no field shows a red marker or message, yet the Next button is disabled. Typing into
the "empty" field by hand and leaving it enables Next again.

## Root cause

Three Granite behaviours combine (foundation clientlib, verified on the Cloud SDK, Sep 2026):

1. **`aria-required="true"` is a validation constraint.** The required validator for
   `input, textarea, select` accepts `element.required`, the `required` attribute *or*
   `aria-required="true"`; Coral's select and autocomplete have matching validators. Adding the
   attribute "for screen readers" turns the field into a gate.
2. **A step check caches verdicts silently.** On any field's `foundation-validation-valid`
   event the wizard runs `step.adaptTo("foundation-validation-helper").isValid()`. Fields not yet
   validated are checked with `suppressEvent: true`: the verdict is stored (`isValidated` true),
   but no event fires and no UI updates. A field that is empty *at that instant* (e.g. waiting
   for an async prefill) is now cached invalid with nothing visible.
3. **Programmatic `input` revalidation is one shared debounce.** Granite revalidates on `change`
   synchronously per element, but on `input` through a single 500 ms debounce whose target is the
   *last* element that fired. Two back-to-back programmatic writes with `.trigger("input")`
   revalidate only the second one. The first keeps its stale verdict, and every later valid event
   from any field re-reads it and disables Next again.

## Fix

After a programmatic write, trigger `change`, not `input`:

```js
$field.val(next).trigger("change");
```

If the gate itself is unwanted, remove `aria-required` rather than working around validation.

## How to spot it next time

```js
const s = $(field).adaptTo("foundation-validation").getValidity();
s.isValidated(), s.isValid()          // true, false while the value is present = stale verdict
$(step).data("foundation-wizard.internal.invalids")   // empty: nothing ever fired an invalid event
```

Trace who toggles the button by wrapping `Granite.$.fn.prop` for `("disabled", …)` on
`.foundation-wizard-control` and recording `new Error().stack`; the stale case shows only
`enableNext` calls with a false answer, never a `foundation-validation-invalid` handler.