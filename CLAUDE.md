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
3. Title the doc by what it covers, drawn from its section headings. Do not append an AEM tie-in such as "and what it can and cannot tell you about AEM"; hardware, protocols and commands do not "tell you about AEM". The AEM relationship is stated once, in a closing "What this means for AEM" section, honestly, including "irrelevant on AEMaaCS" where that is true. `INDEX.md` folder blurbs follow the same rule.

Docs accumulate verified, often counter-intuitive findings (decompiled source, reproduced bugs, confirmed-on-version behavior) rather than restating official docs — entries typically cite how/where something was verified and note version specifics, since AEM behavior varies significantly across 6.5/AMS vs AEMaaCS and across versions.

## Privacy rule (strict, no exceptions)

This repo is shared and version-controlled. **Nothing in it may identify a person, a household, a client, or a specific environment.** Before writing or amending any doc, scrub:

- **People and places**: names, emails, usernames, home-directory paths (`/Users/<name>`), cities, countries, "home connection", employer or client names.
- **Clients and tenants**: client/customer names, their domains, ticket or case numbers, Cloud Manager program/environment IDs (`p12345`, `e67890`), tenant names, internal hostnames and URLs.
- **Network identity**: real public IPs (yours, your ISP's, a client's), ISP hostnames and AS numbers, real MAC addresses or OUI prefixes of your devices, Wi-Fi SSIDs. Only well-known public infrastructure that the doc is *about* (e.g. Adobe/Fastly anycast ranges) may stay.
- **Timestamps of your activity**: no dates at all for when *you* ran, read, probed or verified something, not even month and year. Write "verified on macOS" or "docs-based", never "read Oct 2026". Dates that belong to the subject (an Adobe page revision, an EOL date, a release) are fine.
- **Secrets**: tokens, cookies, passwords, licence keys, private keys, session IDs. Never, even redacted-looking ones.

Replace, don't just delete, so examples stay runnable: documentation ranges `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`; MAC prefix `00:00:5e:00:53:xx` (RFC 7042); AS numbers `64496–64511`; hostnames under `example.com`/`example.net`; "a client", "a consumer ISP connection", "a small test LAN".

Before finishing any edit, grep the changed files for: `20\d\d-\d\d-\d\d`, `\b(\d{1,3}\.){3}\d{1,3}\b`, `([0-9a-f]{1,2}:){5}`, `/Users/`, `@`, and any proper noun that is not a vendor or product. If a verification detail cannot be stated without one of these, state the finding without the detail.
