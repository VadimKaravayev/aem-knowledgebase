# OSGi `Import-Package` "Cannot be resolved" — relaxing bnd's auto-generated version range

When an OSGi bundle won't start on AEM with an error like:

```
org.apache.commons.lang3,version=[3.20,4) -- Cannot be resolved
```

…the bundle is asking the OSGi container for a package version range that **no exported package in AEM can satisfy**. The container exports an *older* version of that package than the range demands. The fix is almost always to **override the import's version constraint** in the bnd instructions, not to change code.

---

## The shopping-list model

The mental model that makes everything below obvious, and the one to explain this with to anyone who doesn't already live in OSGi.

A bundle borrows code from AEM the way a recipe borrows ingredients from a kitchen. So every bundle ships a **shopping list** — the `Import-Package` header in its manifest — saying "to run, I need these packages, at these versions." On startup AEM reads the list and checks its pantry. If even one line can't be filled, it refuses to start the bundle **at all**: not a warning, not degraded mode — the bundle sits in `Installed` and nothing in it runs.

Two kitchens have to be satisfied from one branch, and they stock different versions:

- **AEM 6.5** — on-premise, older stock
- **AEM as a Cloud Service** — Adobe-hosted, newer stock

Three consequences worth internalizing:

1. **The list is generated, not written.** bnd builds it from what the bytecode touches, and picks the version from whatever it found at *compile* time — i.e. the Cloud SDK. So the list is written against the newer kitchen by default, and 6.5 is the one that breaks.
2. **A too-new version request fails as hard as a missing package.** "2.6 or newer" against a pantry holding 2.5 is a refusal, even though 2.5 does the job perfectly.
3. **Unit tests cannot catch any of this.** Tests run with every ingredient already on the table; the list is only checked when a real AEM starts the bundle. A green build says nothing about whether the bundle resolves. Read the generated manifest, then deploy to both kitchens.

That last point is the whole reason this class of bug reaches production: `mvn test` is 100% green and the bundle is dead on arrival.

---

## Why bnd generates a range that AEM can't satisfy

The `bnd-maven-plugin` (and `maven-bundle-plugin`) compute `Import-Package` automatically. For each package your code references, bnd applies its **version policy** to the version of that package present on the *compile* classpath:

- Default **consumer** policy is `[==,+)` → for a package seen at `3.20`, bnd emits `[3.20,4)`.
- So if your build resolves commons-lang3 **3.20** at compile time, every bundle importing it gets `[3.20,4)`.

AEM as a Cloud Service exports many common libraries from the platform (`org.apache.commons.lang3`, `commons.io`, `commons.collections4`, Jackson, etc.) but at **specific, often older versions** (commons-lang3 has been ~3.12–3.14 in recent SDKs, well below 3.20). The bundle demands `>= 3.20`; the container offers `~3.14`; resolution fails.

Crucially the version that drives the range is whatever lands on the compile classpath — frequently a **transitive dependency** of some library you embed, not something you declared. (E.g. an embedded third-party SDK that itself depends on commons-lang3 3.20.)

---

## The fix — pin the import to `version=0.0.0`

Override the import in the bnd `Import-Package` directive so it accepts **any** version the platform exports:

```xml
<bnd><![CDATA[
Import-Package: javax.annotation;version=0.0.0,org.apache.commons.lang3.*;version=0.0.0,*
]]></bnd>
```

`version=0.0.0` is bnd shorthand for "any version" (effectively `[0.0.0,∞)`). It discards bnd's computed `[3.20,4)` and lets the bundle bind to AEM's exported copy. The trailing `*` keeps bnd's automatic handling for every other package.

This is the **established pattern** for platform-provided libraries — list each one with `version=0.0.0`. A real connector bundle's directive that imports a whole set this way:

```
Import-Package: javax.annotation;version=0.0.0,
  com.adobe.granite.translation.core;version=0.0.0,
  org.apache.commons.io.*;version=0.0.0,
  org.apache.commons.lang3.*;version=0.0.0,
  org.apache.commons.collections4.*;version=0.0.0,
  com.day.cq.wcm.api;version=0.0.0,
  org.apache.sling.event.jobs.consumer;version=0.0.0,
  *
```

### Alternative: lock to the major version

`version="[3,4)"` accepts any 3.x but rejects a future 4.x with breaking changes — marginally safer in theory. In practice `version=0.0.0` is the common convention; pick one and stay consistent across the codebase.

---

## When to relax the range vs embed the library

| Approach | What it does | Use when |
|---|---|---|
| **Relax import range** (`version=0.0.0`) | Bind to AEM's platform-exported copy | The library **is** exported by AEM (commons-lang3/io/collections4, Jackson, Granite/WCM packages, Sling APIs). Smallest bundle. **Default choice.** |
| **Embed / `Private-Package`** (`-includeresource: @lib.jar` or `lib.jar;lib:=true`) | Ship your own copy inside the bundle, no import emitted | AEM does **not** export the library at all (e.g. jsoup, a niche SDK). Self-contained but larger; risks classloader duplication if you embed something AEM also exports. |

A bundle typically does **both**: embed only the few libs AEM lacks, and import everything else from the platform with `version=0.0.0`.

```
-includeresource: jsoup*.jar;lib:=true, org.apache.sling.servlet-helpers*.jar;lib:=true
```

---

## The transitive-dependency trap (the important gotcha)

When you **embed a third-party jar** into your bundle (via `-includeresource` / `Private-Package`), every one of *that jar's* transitive dependencies becomes an `Import-Package` the container must satisfy. commons-lang3 is often just the **first** failure — Jackson, an HTTP client, and others can fail the same way once the first is fixed.

So after fixing one unresolved import:

1. Rebuild and redeploy the bundle.
2. If it still won't resolve, read the **next** missing `Import-Package` from the Felix/OSGi console (Web Console → Bundles → the failed bundle shows unsatisfied requirements).
3. Apply the same `version=0.0.0` treatment (if AEM exports it) or embed it (if it doesn't).

Repeat until the bundle is `Active`. Inspecting the generated `target/classes/META-INF/MANIFEST.MF` `Import-Package` header against what AEM exports up front saves round-trips.

---

## An explicit entry is *forced* into the manifest, even when nothing uses the package

The counter-intuitive one, and a trap when **removing** a dependency rather than adding one.

An explicit `Import-Package` entry is not a note *about* a package bnd would have imported anyway — bnd puts it on the list **whether or not any class references it**. Delete every last usage of a package from the source and the manifest still demands it, because the pom told it to.

So a deprecation-removal task has two halves, and the code half is the easy one:

1. Delete the code that uses the package. (Necessary, and by itself achieves nothing at runtime.)
2. **Delete its line from the bnd `Import-Package` directive.** This is the half that actually removes the requirement.

Stop after step 1 and the bundle still declares a hard dependency on a package Adobe is deleting — so it fails to resolve the day the platform drops it, over something the code no longer touches. `grep` over `src/` will look clean and prove nothing; the manifest is the only source of truth:

```bash
# the real check after removing a dependency — not grep over src/
tr ',' '\n' < core/target/classes/META-INF/MANIFEST.MF | grep '<package>'
```

The same asymmetry cuts the other way when adding one: a package reached only through a **new** import (see the confirmed instance below) gets its version range auto-computed from the Cloud SDK and silently becomes 6.5-incompatible. Removing needs a pom edit; adding needs a pom edit. Only the middle case — a package already pinned `0.0.0` — is free.

---

## Diagnostic recipe

```bash
# Where does the offending version come from? (often transitive)
mvn -pl core dependency:tree

# What range did bnd actually emit?
grep Import-Package core/target/classes/META-INF/MANIFEST.MF

# Rebuild + redeploy just the bundle
mvn clean install -pl core -PautoInstallBundle
```

In the OSGi Web Console (`/system/console/bundles`), a bundle stuck in **Installed** (not **Active**) with an "unresolved constraint" / "Cannot be resolved" message is this exact class of problem.

---

## Confirmed instances

### `org.apache.jackrabbit.util` — 2.6 on Cloud vs 2.5 on 6.5 (translated.com connector, Sept 2026)

Reimplementing `ReferenceSearch.adjustReferences` in-house (see
[reference-adjustment-without-referencesearch.md](reference-adjustment-without-referencesearch.md))
introduced the first use of `org.apache.jackrabbit.util.Text` in the bundle. bnd auto-imported it
through the trailing `*` and emitted:

```
org.apache.jackrabbit.util;version="[2.6,3)"
```

That `2.6` is **not** the artifact version — it is the package's own `@Version` annotation, read from
`org/apache/jackrabbit/util/package-info.class` on the compile classpath. And the two platforms
declare different values for it:

| jar | `org.apache.jackrabbit.util` declares |
|---|---|
| `aem-sdk-api` 2026.8.27830 | `2.6.0` |
| `uber-jar` 6.5.23 | **`2.5.0`** |

So `[2.6,3)` is unsatisfiable on 6.5 — every on-premise instance would leave the bundle in
`Installed`. Fixed with `org.apache.jackrabbit.util;version=0.0.0`, matching
`org.apache.jackrabbit.commons;version=0.0.0`, which was already pinned on that same line for
`JcrUtils` — **same library, same trap, different package**. A library already pinned is no
protection for its sibling packages.

Check a package's declared version rather than guessing from the artifact version:

```bash
unzip -o -q "$JAR" 'org/apache/<pkg path>/package-info.class' -d /tmp/pi
javap -v /tmp/pi/org/apache/<pkg path>/package-info.class | grep -E 'Utf8 +[0-9]+\.[0-9]'
```

The same task also hit the removal half of the trap above: after deleting every reference to
`com.day.cq.wcm.commons`, the generated manifest still demanded it, because its explicit
`version=0.0.0` line was still in the pom.

### commons-lang3 3.20 via an embedded SDK (Crowdin connector, June 2026)

Crowdin AEM connector (`core` bundle), June 2026. The bundle embedded `crowdin-api-client-java` via `-includeresource: @crowdin-api-client-java-*.jar`; that SDK pulled commons-lang3 **3.20** transitively, so bnd emitted `org.apache.commons.lang3;version=[3.20,4)`, which the AEM 2026.6 SDK (exporting an older 3.x) could not resolve. Fixed by adding `org.apache.commons.lang3.*;version=0.0.0` to the bnd `Import-Package` directive — matching the convention already used in the sibling `devhandler-aem-plugin` connector.

---

## Reference

- bnd version policies: <https://bnd.bndtools.org/chapters/170-versioning.html>
- bnd `Import-Package` instruction: <https://bnd.bndtools.org/heads/import_package.html>
- AEM exported packages vary by SDK version — verify against the running instance, not from memory.