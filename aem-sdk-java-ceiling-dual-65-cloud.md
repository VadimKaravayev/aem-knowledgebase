# The Java ceiling of dual AEM 6.5 + Cloud support (one branch)

## The pattern

A connector/plugin codebase that supports **AEM 6.5 on-prem and AEMaaCS from a single branch** must compile to the lowest common Java baseline — **Java 11** for classic 6.5 (6.5 LTS is a different story, but classic 6.5 installs are the constraint). That Java ceiling silently becomes an **`aem-sdk-api` version ceiling**: newer SDK releases transitively pull dependencies compiled for Java 17+, so the build breaks even though your own code never changed.

Concrete case (verified on the translated.com connector, Aug 2026): bumping `aem-sdk-api` to **2026.7.27083** pulls in a Jackrabbit Oak compiled for Java 17 (class file version 61). On a Java 11 toolchain the build fails on the SDK's own classes.

> **Two refinements (Sept 2026).** (1) The baseline can be *lower* than 11: classic 6.5 supports Java 8 as well, so a customer running 6.5 on a **JVM 8** pushes the floor to Java SE 8 — ask which JVM they actually run rather than assuming 11. (2) This page frames the ceiling as forcing an old *toolchain*, but that is only true if you compile **on** the old JDK. Keep building on a new JDK (21) and target the old platform with `--release N`, and the SDK ceiling disappears — the build reads the Java 17+ SDK classes fine while still emitting old bytecode. See [java-target-platform-vs-build-jdk.md](java-target-platform-vs-build-jdk.md) for the build-JDK / target-platform / class-file-version / runtime-JVM distinction, why `-source`/`-target` is not enough, and the verification recipe.

## Why this bites: deprecation migrations become doubly blocked

Adobe's Cloud Manager code scan (rule `java:S1874`, tag `obsolete`) flags deprecated-package usage with a removal deadline. The prescribed replacement often exists **only in newer Cloud releases** — exactly the SDK versions the Java ceiling forbids.

Case in point: `com.day.cq.wcm.commons.ReferenceSearch` (deprecated 2026-07-01, removal 2027-03-31) → replacement `com.day.cq.wcm.commons.utils.ReferenceSearch`, first shipped in SDK 2026.7.27083. For a dual-support project the official migration is blocked twice over:

1. **Compile time:** the SDK containing the replacement needs Java 17.
2. **Runtime on 6.5:** the replacement package is never exported by 6.5, so a single-branch bundle importing it would fail OSGi resolution on-prem even if it compiled. Verified by `unzip -l` (Sept 2026): `com/day/cq/wcm/commons/utils/ReferenceSearch.class` is absent from uber-jar 6.5.23 and present in aem-sdk-api 2026.8.27830. Full matrix in [reference-adjustment-without-referencesearch.md](reference-adjustment-without-referencesearch.md).

The removal deadline only applies to Cloud (6.5 keeps the old class), but a shared branch has to satisfy both.

## Escape hatches, ranked

1. **Reimplement the small API surface you actually use.** Deprecated product classes are often used at a single call site for a fraction of their functionality. `ReferenceSearch.adjustReferences(Node, from, to)` is the easy half of that class: no queries, just a subtree walk rewriting `String`/`String[]` properties that reference the `from` path — ~40–60 lines of plain JCR, unit-testable with AEM Mocks. **Trap for reimplementers:** match path *boundaries*, not raw prefixes (`/content/site/en` must not corrupt `/content/site/en-gb`), and cover paths embedded inside longer strings (rich-text `href`s), both of which the stock implementation handles. The repo-wide `search()` half is the hard part — but you rarely need to reimplement it either: the non-deprecated `ReferenceProvider` service covers discovery, which is what `search()` was being used for. Worked recipe for both halves: [reference-adjustment-without-referencesearch.md](reference-adjustment-without-referencesearch.md).
2. **Split the Java baseline** — separate branches/artifacts for 6.5 (Java 11) and Cloud (Java 17+). Solves everything, costs you the single-branch model dual-support projects exist to preserve.
3. **Wait it out** is only viable until the removal date, and only pushes the same decision closer to the deadline.

Not an option: just switching the import to the `commons.utils` package "for the scan" — see the runtime-on-6.5 block above.

## Recognizing the failure

- Compiling against the newer SDK on JDK 11: `javac` errors of the form `class file has wrong version 61.0, should be 55.0` on SDK-internal (often Oak) classes — the version numbers name the class-file versions (61 = Java 17, 55 = Java 11).
- The fix-by-bumping-JDK reflex is the trap: check first whether the artifact still needs to run on classic 6.5.