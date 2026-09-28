# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A plain-markdown knowledgebase of AEM (Adobe Experience Manager) patterns, gotchas, and recipes — no build, lint, or test tooling; the repo is the content itself. It's kept central so multiple AEM projects (and the Claude Code agents working on them) pull from one source of truth instead of re-deriving the same answers per project.

## Structure

- `INDEX.md` — the entry point: one line per doc, summarizing its finding densely enough to often answer a question without opening the file.
- One topic per file, kebab-case filenames (e.g. `touch-ui-granite/dialog-showhide-fields.md`), grouped into topic folders:
  - `ai-integration/` — LLM and external service integration with AEM.
  - `assets-dam/` — AEM Assets and DAM: image delivery, Dynamic Media, Brand Portal.
  - `cloud-service/` — AEM as a Cloud Service platform and Cloud Manager.
  - `dispatcher-url-delivery/` — Dispatcher, CDN, URL resolution and the request path.
  - `infrastructure-ops/` — AEM 6.5 / AMS infrastructure, runmodes, upgrades, diagnostics.
  - `java-osgi-build/` — Java, OSGi, Maven build, testing, servlet patterns.
  - `sites-content/` — AEM Sites content, references, headless delivery, Core Components.
  - `touch-ui-granite/` — Touch UI, Granite and Coral dialogs, pickers, consoles.
  - `translation-i18n/` — Translation connectors, i18n dictionaries, language handling.
- Filenames are unique across all folders. Cross-references use wiki links `[[name]]` (filename without folder or extension, so they survive moves) or relative markdown links.
- Each doc is self-contained: written so a reader can act on just that one file without pulling in the rest of the folder.

## Workflow

**Answering an AEM question**: check `INDEX.md` first for an existing doc before researching from scratch.

**Adding a doc**:
1. Create a kebab-case markdown file in the folder that fits the topic best (create a new folder only when several docs would share it).
2. Add a one-line entry under that folder's section in `INDEX.md` summarizing the finding (see existing entries for the expected density — key facts, gotchas, and the exam/design trap if there is one, not a generic teaser).

Docs accumulate verified, often counter-intuitive findings (decompiled source, reproduced bugs, confirmed-on-version behavior) rather than restating official docs — entries typically cite how/where something was verified and note version specifics, since AEM behavior varies significantly across 6.5/AMS vs AEMaaCS and across versions.