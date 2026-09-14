# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A plain-markdown knowledgebase of AEM (Adobe Experience Manager) patterns, gotchas, and recipes — no build, lint, or test tooling; the repo is the content itself. It's kept central so multiple AEM projects (and the Claude Code agents working on them) pull from one source of truth instead of re-deriving the same answers per project.

## Structure

- `INDEX.md` — the entry point: one line per doc, summarizing its finding densely enough to often answer a question without opening the file.
- One topic per file at the root, kebab-case filenames (e.g. `dialog-showhide-fields.md`).
- Each doc is self-contained: written so a reader can act on just that one file without pulling in the rest of the folder.

## Workflow

**Answering an AEM question**: check `INDEX.md` first for an existing doc before researching from scratch.

**Adding a doc**:
1. Create a kebab-case markdown file at the root.
2. Add a one-line entry to `INDEX.md` summarizing the finding (see existing entries for the expected density — key facts, gotchas, and the exam/design trap if there is one, not a generic teaser).

Docs accumulate verified, often counter-intuitive findings (decompiled source, reproduced bugs, confirmed-on-version behavior) rather than restating official docs — entries typically cite how/where something was verified and note version specifics, since AEM behavior varies significantly across 6.5/AMS vs AEMaaCS and across versions.