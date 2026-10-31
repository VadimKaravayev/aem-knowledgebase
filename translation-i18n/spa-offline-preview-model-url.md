# AEM SPA page blank in an offline preview: model URL derived from the location

- **Symptom:** an AEM SPA Editor page (WKND SPA React) packaged as a static preview renders blank; the console shows a failed fetch of a strange URL such as `…/home.model.phrase-preview/index.html.json/index.html`. The same package works when unpacked into another folder.
- **Root cause:** besides `cq:pagemodel_root_url`, the SPA's `ModelManager` derives a model URL from `window.location`, inserting `.model` after the first dot in the path. Served from anywhere but AEM, that URL is whatever the location makes it, so a fetch shim matching `endsWith(".model.json")` lets it through to a real fetch that fails.
- **Fix:** the shim answers every URL containing `.model.` with the one model it carries (a preview has exactly one), as `new Response(model)`, and never fetches: a page opened from `file://` cannot fetch at all (origin `null`, CORS-blocked).
- **How to spot it:** blank page whose failing fetch URL has `.model` spliced into a folder name; works in a folder without a dot, fails in one with a dot.
