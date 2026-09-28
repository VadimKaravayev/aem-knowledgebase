# Cloud Manager pipeline steps — where failures surface

The Non-Production and Production Deployment pipelines run a fixed sequence of steps; knowing what each step actually *does* narrows down where to look when one fails.

Rough order (Non-Production pipeline; Production pipeline is similar but adds approval gates):

1. **Validate** — static code quality scan (Cloud Manager code quality rules).
2. **Build** — Maven build of the project (`all`, `core`, `ui.apps`, etc.).
3. **Unit Tests** — runs the project's unit test suite.
4. **Build Images** — packages the built artifact with the AEM base release into Author/Publish container images. Critically, this step **boots a transient AEM instance** to run repository initializers (repoinit), install content packages, and build indexes as part of image assembly — it is not just a static packaging step.
5. **Deploy** — rolls the built images out to the target environment (rolling deploy: old and new code briefly serve traffic together; see [rolling-deployment-two-version-overlap.md](rolling-deployment-two-version-overlap.md)).
6. **Functional/UI Tests, Security Scan, Performance Test** — post-deploy validation steps (environment/program dependent).
7. **Approval** (Production pipeline only) — manual gate before proceeding to production deploy.

**Why this matters for triage:** because Build Images actually starts a real repository, a `SlingRepositoryInitializer` exception (bad repoinit ACL/group statement, malformed CND, etc.) aborts *at this step*, not at Deploy — the repo never comes up, so the pipeline fails before an image is even produced. This is a common exam/triage trap: a stack trace mentioning JCR repository startup during "Build Images" is a build-time repoinit failure, not a runtime/deploy issue. See `question-32.md` in this repo for a worked example (repoinit `contributor is not a group` vs. two unrelated but non-fatal log entries — a benign `/libs` Closure Compiler SEVERE message and a vanity-path traversal WARN — that are easy to misdiagnose as the cause).

Standard first triage steps for a previously-green pipeline that fails at Build Images: (1) check recent commits to repoinit/ACL config in the pipeline branch, (2) re-run the pipeline once to confirm the failure is deterministic before making changes.
