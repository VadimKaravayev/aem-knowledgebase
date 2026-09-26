# `<template>` rows cloned with jQuery never become Coral elements

- **Symptom:** rows cloned from a `<template>` holding `<tr is="coral-table-row">` render as bare HTML: no `_coral-Table-row`/`_coral-Table-cell` classes, ~26px rows, no frame, header labels misaligned with the cells. No error.
- **Root cause:** `template.content` belongs to an inert document. `$(template.content).children().clone()` (or `cloneNode`) keeps that owner, and the customized built-in (`is="coral-*"`) is never upgraded after it is appended to the page.
- **Fix:** `document.importNode(template.content, true).firstElementChild`, which creates the copy in the page's document; wrap it in jQuery after that.
- **Spot it:** `row.className` is empty after append; an upgraded row reads `_coral-Table-row`. Verified on AEM SDK (Coral Spectrum), Sep 2026, Phrase connector wizard (`clientlibs/new-project/js/content.js`).