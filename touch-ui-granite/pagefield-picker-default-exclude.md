# Pagefield picker: the default exclude silently empties pickers rooted in system trees

## The gotcha

The stock Granite page picker
(`/mnt/overlay/cq/gui/content/coral/common/form/pagefield/picker.html`) accepts an
`exclude` query parameter (a path regex of subtrees to hide). When the parameter is
**absent**, the picker does not show everything — it falls back to a **built-in default
exclusion list** covering the system subtrees under `/content`:

```
/content/(catalogs|dam|experience-fragments|services|launches|mac|publications|
usergenerated|communities|community-components|community-templates|forms|projects|
phonegap|mobileapps|screens/(svc|blueprints)|entities|versionhistory)(/.*)?
```

Consequence: root the picker **inside** one of those trees — e.g.
`?root=%2fcontent%2fexperience-fragments` to build an Experience Fragment picker — and
every node is filtered out. The dialog opens, the column view renders, and it is
**empty**. No error, no console message; search by path still finds the nodes (search
goes through a different servlet), which makes the bug look intermittent.

## The fix

Pass **any** `exclude` value; an explicit parameter *replaces* the default list rather
than extending it. The established idiom (used by shipping connectors, e.g. the
translated.com AEM plugin) is a path that matches nothing:

```
...pagefield/picker.html?root=%2fcontent%2fexperience-fragments&selectionCount=multiple&exclude=%2fnonexistent
```

`exclude=/nonexistent` looks like a leftover dummy parameter — it is load-bearing. Do
not "clean it up".

## Verifying (60 seconds, no deploy)

Fetch the picker URL with and without the parameter and count collection items:

```js
const base = "/mnt/overlay/cq/gui/content/coral/common/form/pagefield/picker.html?root=%2fcontent%2fexperience-fragments";
for (const url of [base, base + "&exclude=%2fnonexistent"]) {
    const html = await (await fetch(url)).text();
    console.log(url, (html.match(/data-foundation-collection-item-id/g) || []).length);
}
// 0 without exclude, N with
```

## Related notes

- Experience fragments and their variations are plain `cq:Page` nodes, so the pagefield
  picker is the right picker for an XF selector — no dedicated XF picker exists. A
  variation is identified by `jcr:content` carrying `cq:xfVariantType` or
  `cq:xfMasterVariation`; the XF container page has neither.
- The same default-exclude regex is what you pass *explicitly* when building a plain
  page picker that should hide the system trees (the stock Pages picker in site admin
  does exactly that).

Confirmed on the Crowdin connector New Job wizard (local AEM SDK), July 2026.