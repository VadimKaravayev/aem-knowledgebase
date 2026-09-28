# Granite.I18n.get snippets: falsy values silently skip `{0}` substitution

## The gotcha

Client-side `Granite.I18n.get(text, snippets, note)` patches `{0}`/`{1}`… placeholders via an internal `patchText(text, snippets)` whose implementation guards the whole substitution with a plain truthiness check:

```js
var patchText = function(text, snippets) {
    if (snippets) {                       // <-- 0, "", null, undefined: no substitution at all
        if (Array.isArray(snippets)) {
            for (var i = 0; i < snippets.length; i++) {
                text = text.replace("{" + i + "}", snippets[i]);
            }
        } else {
            text = text.replace("{0}", snippets);
        }
    }
    return text;
};
```

So `Granite.I18n.get("References: {0}", count)` works for every count **except 0** — with `count = 0` the user sees the literal text `References: {0}`. Same for an empty string snippet. The bug is invisible in happy-path testing because non-zero counts substitute fine; it surfaces exactly at the "empty state" (all items removed, zero results).

`Granite.Util.patchText` (the public twin) has the identical `if (snippets)` guard.

## The rule

**Always pass snippets as an array**, even for a single value:

```js
Granite.I18n.get("References: {0}", [count])   // works for 0 — array is truthy
```

An array is truthy even when it contains falsy values, and `String.prototype.replace` coerces each element, so `[0]` renders as `"References: 0"`.

This also composes with the i18n best practice that motivates placeholders in the first place: never build localized strings by concatenation (`I18n.get("References: ") + count` breaks word order/pluralization in some languages) — use a single `"References: {0}"` message with the count passed as `[count]`.

## Verification

Confirmed by reading the deployed clientlib on a local author:
`curl -su admin:admin http://localhost:4502/etc.clientlibs/clientlibs/granite/utils.js` — both `Granite.Util.patchText` (~line 248) and the private `patchText` used by `Granite.I18n` (~line 868) carry the `if (snippets)` guard. Reproduced in the translated.com connector's New Order wizard References dialog (counter showed `References: {0}` after removing the last reference), Aug 2026.