# FileVault validation: making packages comply

Since **filevault-package-maven-plugin 1.1.0** every build runs a validator
suite against package content and filters; legacy/migrated packages
routinely fail it. This note = wcm.io's field guide to the common failures.
Filter semantics in [[filevault-workspace-filter]]; the validator-config
merge trap (`combine.self="override"`) and index-specific settings in
[[oak-indexing-aemaacs]]; package-type definitions in
[[aemaacs-content-package-structure]].

---

## Step zero: fix the packageType, not the errors

**Which validations run depends on `packageType`**, so a wrong type
produces misleading errors. `application` (pure `/apps`//`/libs`, whole
subtrees, no filters/subpackages/OSGi), `content` (`/content`//`/conf`,
filters allowed), `container` (only subpackages + OSGi, no content),
`mixed` (legacy catch-all — **avoid**; split the package instead, which is
also the AEMaaCS-readiness move). App content under `/etc` → move to
`/apps` first.

## The two classic errors and their fixes

**`Filter root's ancestor is not covered by any of the specified
dependencies nor a valid root`** — the filter claims `/apps/myapp/...` but
nothing provides `/apps/myapp`. Fix: tell the `jackrabbit-filter` validator
the parent legitimately exists already (AEM standard path, repoinit- or
other-package-created) via `validatorsSettings` →
`jackrabbit-filter.options.validRoots = /apps/myapp`.
**wcm.io's counter-recommendation:** this `validRoots` config achieves what
Adobe's archetype does with the `apps-repository-structure` package +
`repositoryStructurePackages` property, *without* the extra Maven module —
they consider the structure package the more verbose route. (Know both:
Adobe docs still teach the structure package,
[[aemaacs-content-package-structure]].)

**`Given root node name 'xxx:yyy' (implicitly given via filename) cannot be
resolved`** — a folder with an escaped namespace (`_cq_tags`) whose
`.content.xml` is missing the namespace declaration (`xmlns:cq`). Fix: add
the declaration, or re-export the package with current tooling.

## Disabling checks (last resort)

Per-validator `validatorsSettings` entries support `isDisabled = true`
(whole validator) or option-level relaxation; validator names and options
are in the FileVault validation reference
(jackrabbit.apache.org/filevault/validation.html). wcm.io's framing: most
reported violations are *real*; disable only when you're certain it can't
bite at deploy time — and remember disabled-locally still runs in Cloud
Manager's own analysis ([[aem-analyser-maven-plugin]]).

## Exam checklist

- Validators ship with filevault-package-maven-plugin ≥ 1.1.0; **the set
  that runs depends on packageType** — verify the type before debugging.
- Ancestor-not-covered → `validRoots` on `jackrabbit-filter` (lighter
  alternative to the repository-structure package).
- Unresolvable `xxx:yyy` root from a filename → missing namespace
  declaration in `.content.xml`.
- `isDisabled` per validator exists, last resort only.

## References
- [How to make your content packages comply with Jackrabbit FileVault Validation (wcm.io)](https://wcm-io.atlassian.net/wiki/spaces/WCMIO/pages/1353056261/How+to+make+your+content+packages+comply+with+Jackrabbit+FileVault+Validation)
- [FileVault validation reference (Apache Jackrabbit)](https://jackrabbit.apache.org/filevault/validation.html)

wcm.io wiki (read via Confluence API, Oct 2026); XML snippets paraphrased —
exact syntax in the linked pages.
