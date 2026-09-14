# OSGi `Import-Package` "Cannot be resolved" — relaxing bnd's auto-generated version range

When an OSGi bundle won't start on AEM with an error like:

```
org.apache.commons.lang3,version=[3.20,4) -- Cannot be resolved
```

…the bundle is asking the OSGi container for a package version range that **no exported package in AEM can satisfy**. The container exports an *older* version of that package than the range demands. The fix is almost always to **override the import's version constraint** in the bnd instructions, not to change code.

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

## Confirmed instance

Crowdin AEM connector (`core` bundle), June 2026. The bundle embedded `crowdin-api-client-java` via `-includeresource: @crowdin-api-client-java-*.jar`; that SDK pulled commons-lang3 **3.20** transitively, so bnd emitted `org.apache.commons.lang3;version=[3.20,4)`, which the AEM 2026.6 SDK (exporting an older 3.x) could not resolve. Fixed by adding `org.apache.commons.lang3.*;version=0.0.0` to the bnd `Import-Package` directive — matching the convention already used in the sibling `devhandler-aem-plugin` connector.

---

## Reference

- bnd version policies: <https://bnd.bndtools.org/chapters/170-versioning.html>
- bnd `Import-Package` instruction: <https://bnd.bndtools.org/heads/import_package.html>
- AEM exported packages vary by SDK version — verify against the running instance, not from memory.