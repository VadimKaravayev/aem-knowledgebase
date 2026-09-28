# AEM Knowledgebase

Shared notes on AEM (Adobe Experience Manager) patterns, gotchas, and recipes.

Plain markdown, kept central so multiple projects — and the Claude Code agents working on them — can pull from one source of truth instead of reinventing answers per project.

## Layout

- `INDEX.md` — short pointer list of every doc with a one-line hook.
- One topic per file, kebab-case, grouped into topic folders:
  - `ai-integration/` — LLM and external service integration with AEM.
  - `assets-dam/` — AEM Assets and DAM: image delivery, Dynamic Media, Brand Portal.
  - `cloud-service/` — AEM as a Cloud Service platform and Cloud Manager.
  - `dispatcher-url-delivery/` — Dispatcher, CDN, URL resolution and the request path.
  - `infrastructure-ops/` — AEM 6.5 / AMS infrastructure, runmodes, upgrades, diagnostics.
  - `java-osgi-build/` — Java, OSGi, Maven build, testing, servlet patterns.
  - `sites-content/` — AEM Sites content, references, headless delivery, Core Components.
  - `touch-ui-granite/` — Touch UI, Granite and Coral dialogs, pickers, consoles.
  - `translation-i18n/` — Translation connectors, i18n dictionaries, language handling.
- Filenames are unique across folders; wiki links `[[name]]` refer to the filename without folder or extension.

## Adding a doc

1. Create a kebab-case markdown file in the best-fitting topic folder.
2. Add a one-line entry under that folder's section in `INDEX.md`.

Keep each doc self-contained: someone (or an agent) should be able to read just that one file and act on it without needing the rest of the folder.