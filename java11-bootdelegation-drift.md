# `jdk.internal.reflect` `NoClassDefFoundError` — the Java 8 boot-delegation list that survives the JDK upgrade

Symptom, from `error.log` on an AEM 6.5 instance running a Java 11 JVM:

```
*ERROR* [FelixStartLevel] org.apache.sling.scripting.sightly bundle
org.apache.sling.scripting.sightly:1.1.2.1_4_0 (557)
[org.apache.sling.scripting.sightly.impl.engine.extension.use.JavaUseProvider(3410)] :
Error during instantiation of the implementation object
(java.lang.NoClassDefFoundError: jdk/internal/reflect/ConstructorAccessorImpl)
java.lang.NoClassDefFoundError: jdk/internal/reflect/ConstructorAccessorImpl
    at java.base/jdk.internal.misc.Unsafe.defineClass0(Native Method)
    at java.base/jdk.internal.reflect.ClassDefiner.defineClass(ClassDefiner.java:63)
    at java.base/jdk.internal.reflect.MethodAccessorGenerator$1.run(MethodAccessorGenerator.java:400)
```

**The cause is not the build.** Nothing in `pom.xml` is wrong and the JDK is not unsupported — the
bundle compiled, deployed and *resolved*. The instance's OSGi boot-delegation list is still the
Java 8 one, so the container cannot see `jdk.internal.reflect.*`.

---

## Why the JDK upgrade doesn't fix itself

`org.osgi.framework.bootdelegation` lives in `crx-quickstart/conf/sling.properties` — **instance
state on disk, not something derived from the running JVM.** In Java 8 the reflection internals were
`sun.reflect.*`, covered by the long-standing `sun.*,com.sun.*` delegation. In Java 11 they moved to
`jdk.internal.reflect.*`, which that list does not match, so Adobe added explicit entries to the
newer quickstart templates.

Verified across three instances on one machine (Sept 2026):

| Instance | `quickstart.build` | `org.osgi.framework.bootdelegation` |
|---|---|---|
| AEM 6.5.0 GA | `0.0.0.0_0_0_6_5_.20190328` | `sun.*,com.sun.*` |
| AEM 6.5 LTS | `6.6.2.20260216` | `sun.*,com.sun.*,jdk.internal.reflect,jdk.internal.reflect.*` |
| Cloud SDK | `2026.7.27083` | `sun.*,com.sun.*,jdk.internal.reflect,jdk.internal.reflect.*` |

Note the newer value carries **both** the bare package and the wildcard.

The 6.5.0 GA row is a live reproduction of the fault: that instance runs `OpenJDK 64-Bit Server VM
11.0.25` (per its own `SlingMainServlet` startup line in `error.log`), has been started and stopped
repeatedly under it, and `sling.properties` has a *later* mtime than the conf directory — so the file
is rewritten on startup and the stale value still persists. **Restarting under JDK 11 does not heal
it.** Whether the launcher preserves the existing value or recomputes it from the 6.5.0-era template
is not distinguishable from the filesystem, and doesn't matter: either way the entries never appear.

### Don't expect a conditional entry to add them

`sling.properties` also holds conditional-looking keys — on the 6.5.0 instance:

```
sling.bootdelegation.sun=sun.*,com.sun.*
sling.bootdelegation.ibm=com.ibm.xml.*
sling.bootdelegation.weblogic=weblogic.xml.*
sling.bootdelegation.jboss=__redirected
sling.bootdelegation.jboss.__redirected=__redirected
```

On 6.5.0 the flat property equals `sling.bootdelegation.sun` exactly, so the flat key is written back
as the computed merge of these. But on LTS and Cloud, `sling.bootdelegation.sun` is *still* only
`sun.*,com.sun.*` while the flat key has the JDK-11 entries — **nothing keys `jdk.internal.reflect`
to a `sling.bootdelegation.*` entry.** It is a plain default in the newer template. So there is no
conditional mechanism that will notice the JVM changed and add it for you.

---

## How an instance ends up stale

`conf/` is written at first unpack and then treated as instance state:

1. **Reused `crx-quickstart` folder** — unpacked and first started under JDK 8, then `JAVA_HOME`
   switched to JDK 11 and the *same* folder restarted. The classic local-dev case, and the one
   reproduced above.
2. **In-place upgrade / service pack** — an SP installs packages into the repository; it does not
   regenerate `conf/`. The AEM version now supports Java 11, the file on disk predates it.
3. **Hand-edited `sling.properties`** — teams routinely edit this file for extra delegation or Felix
   tuning. Once modified it won't be clobbered, so a newer quickstart's entries never land. The
   nastiest variant, because everything else about the instance looks current.
4. **Cloned environments** — on AMS especially, environments are built from a snapshot or golden
   image of an older instance; `conf/` is inherited wholesale.
5. **A start script pinning `-Dorg.osgi.framework.bootdelegation=...`** — a system property overrides
   the file entirely, and start scripts get copied forward across versions unchanged.

---

## Why it fails at *instantiation*, not at bundle resolution

Two delays stack up, and together they are why this reaches production:

- **Boot delegation is consulted lazily.** It only matters when a classload escapes the bundle's
  wiring. Every bundle resolves and starts clean; the failure waits for the first request that
  reflectively constructs something — here Sightly's `JavaUseProvider` building a Use-object.
- **Reflection inflation delays it further.** HotSpot serves the first `sun.reflect.inflationThreshold`
  (default 15) invocations of a given accessor with a native one, then generates a bytecode accessor
  that *extends* `jdk.internal.reflect.ConstructorAccessorImpl` and defines it via `Unsafe.defineClass`.
  Only that generated class needs the delegated package. This is standard JDK behavior, and it matches
  the stack exactly (`MethodAccessorGenerator` → `ClassDefiner.defineClass` → `Unsafe.defineClass0`).

So the component renders correctly the first several times and then starts throwing — the symptom
looks load-related or intermittent, which sends people hunting in the wrong place.

---

## Fix

Diff `crx-quickstart/conf/sling.properties` against a freshly unpacked quickstart of the target
version (`java -jar aem-quickstart.jar -unpack`) and append the missing entries:

```properties
org.osgi.framework.bootdelegation=sun.*,com.sun.*,jdk.internal.reflect,jdk.internal.reflect.*
```

Restart. Do not touch the build — no `pom.xml`, bnd or dependency change addresses this.

Quick triage across a fleet:

```bash
grep -H '^org.osgi.framework.bootdelegation' */crx-quickstart/conf/sling.properties
```

Any instance showing `sun.*,com.sun.*` alone while running a Java 11+ JVM is a latent failure.

---

## Related

- [aem-sdk-java-ceiling-dual-65-cloud.md](aem-sdk-java-ceiling-dual-65-cloud.md) — the *compile*-side
  Java-version constraints for a dual 6.5/Cloud branch; this doc is the *runtime* side.
- [java-target-platform-vs-build-jdk.md](java-target-platform-vs-build-jdk.md) — build JDK vs target
  platform, the distinction that rules out "unsupported JDK" as the cause here.
- [osgi-import-package-version-range.md](osgi-import-package-version-range.md) — the other OSGi
  visibility failure mode: an import the container can't satisfy, which fails at *resolution* rather
  than instantiation.
