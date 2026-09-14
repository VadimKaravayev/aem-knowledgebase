# AEM Knowledgebase

Shared notes on AEM (Adobe Experience Manager) patterns, gotchas, and recipes.

Plain markdown, kept central so multiple projects — and the Claude Code agents working on them — can pull from one source of truth instead of reinventing answers per project.

## Layout

- `INDEX.md` — short pointer list of every doc with a one-line hook.
- One topic per file at the root, named in kebab-case (e.g. `dialog-showhide-fields.md`).

## Adding a doc

1. Create a kebab-case markdown file at the root.
2. Add a one-line entry to `INDEX.md`.

Keep each doc self-contained: someone (or an agent) should be able to read just that one file and act on it without needing the rest of the folder.