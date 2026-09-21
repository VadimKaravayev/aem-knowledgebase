# Pre-launch testing types (AEM Architect exam)

Five testing types come up together on exam scenarios about launch readiness. Two are about discovering *unknown weaknesses*; three are about confirming *known behavior* still works.

**Load Testing** — Simulates expected/peak concurrent traffic against the platform. Surfaces performance/infra weaknesses: undersized author/publish heap, dispatcher cache misconfiguration, slow JCR queries, insufficient CDN offload, under-provisioned instance counts.

**Penetration Testing** — Actively attempts to exploit vulnerabilities (the "ways to exploit the application" phrasing). On AEM: exposed `/crx/de` or `/system/console`, default admin credentials, dispatcher filter bypasses exposing `/bin` or `.json`/`.infinity.json` selectors, CSRF gaps, XSS in components, insecure OSGi configs.

**Unit Testing** — Tests individual code units in isolation (a Sling Model, servlet, utility class), typically mocked (AEM Mocks / wcm.io), no running instance needed. Verifies logic correctness at the smallest scope, done continuously during development.

**Regression Testing** — Re-runs existing test suites after a code change to confirm nothing previously working broke. Not about finding new weaknesses — a safety net for changes (upgrades, hotfixes, releases).

**Functional Testing** — Verifies the application does what the business requirements say (does search return results, can a user submit a form). Validates behavior against spec, not performance or security.

## Exam pattern

A scenario combining "identify system weaknesses" + "ways to exploit the applications" → answer is **Load Testing + Penetration Testing**, not Unit/Regression/Functional (those confirm known behavior rather than discover unknown weaknesses).

Source: [question-22.md](../my-own-repos/aem-architect-lab/questions/question-22.md)
