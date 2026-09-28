# Targeting an old Java platform from a new build JDK (AEM connectors)

## The three axes people conflate

"We support Java 8" is ambiguous. There are three independent things, and mixing them up produces bugs that only appear on the customer's instance:

| Axis | Example value | Governs | Failure if violated |
|---|---|---|---|
| **Build JDK / toolchain** | JDK 21 | which `javac` runs | build won't start / can't read newer SDK classes |
| **Target platform** (`--release N`) | Java SE 8 | language level + **API surface** + class file version | see the two rows below |
| **Class file major version** | 52 (`0x0034`) | what the JVM's verifier accepts | `UnsupportedClassVersionError` **at class load** |
| **Class library (JDK N API)** | JDK 8 `java.*` | which methods actually exist at runtime | `NoSuchMethodError` / `NoClassDefFoundError` **at first call** |
| **Runtime JVM** | JVM 8 (HotSpot 8) | where it executes | — |

Accurate phrasing: *compiled **with** JDK 21, **for** Java SE 8, producing class file version 52, running on **JVM 8** and upward.* You never compile *on* the old JDK — that would re-introduce the SDK ceiling in [aem-sdk-java-ceiling-dual-65-cloud.md](aem-sdk-java-ceiling-dual-65-cloud.md).

Naming: **Java SE 8 == 1.8**. JDK 8 shipped as `1.8.0_x`, and OSGi still spells it `JavaSE 1.8`, so the manifest says `1.8` while `--release` takes `8`.

## Use `--release`, never `-source`/`-target` alone

This is the whole point. `-source 8 -target 8` pins the **language level** and the **class file version**, but still compiles against the *build JDK's* class library. So:

```java
List.of("a")          // compiles clean, bytecode is version 52
```

…and then throws `NoSuchMethodError: java.util.List.of` on a JVM 8, because that method does not exist in the JDK 8 library. `--release 8` compiles against JDK 8's historical API signatures (shipped in the JDK's `ct.sym`) and rejects it at build time.

**The class file version being correct proves nothing about API compatibility.** They are separate failure modes with separate symptoms (load time vs first call).

## The `invokedynamic` string-concat trap

The subtlest instance of the above. From Java 9, javac compiles `"a" + x` to an `invokedynamic` against `java.lang.invoke.StringConcatFactory` instead of `StringBuilder`. That class does not exist on JVM 8, so such a bundle fails at **link** time on the customer's instance while the class file version looks perfectly correct.

`--release 8` falls back to `StringBuilder`. Verify with `javap -c`:

```
javap -c -p -cp target/classes com.example.Foo | grep -cE "makeConcat|StringConcatFactory"   # must be 0
```

This matters for **generated** code you never wrote — Lombok's `@Value`/`@Data` `toString()` is string concatenation, so it is exactly the shape that would break. (Verified Sept 2026: Lombok 1.18.46 under `--release 8` emits `StringBuilder`, 0 `StringConcatFactory` refs; `@Value @Builder @Jacksonized @Slf4j @NonNull` all compile to class file 52. The `StringBuilder` fallback comes from `--release`, not from Lombok.)

## Maven wiring, and the trap in the AEM archetype

Set the target as a **property** and have the compiler plugin read it:

```xml
<properties>
  <maven.compiler.release>8</maven.compiler.release>
</properties>
...
<plugin>
  <artifactId>maven-compiler-plugin</artifactId>
  <configuration>
    <release>${maven.compiler.release}</release>
  </configuration>
</plugin>
```

The AEM Project Archetype (v56 checked) hard-codes `<release>11</release>` **directly on the plugin**, which silently overrides any `-Dmaven.compiler.release=` on the command line. Symptom: you pass `-Dmaven.compiler.release=8`, the build reports SUCCESS, and still emits class file 55. Always verify the emitted bytecode rather than trusting the flag.

## Verification recipe (the only thing that actually proves it)

```bash
# 1. every emitted class is version 52
find core/target/classes -name '*.class' -exec sh -c 'xxd -p -s 6 -l 2 "$1"' _ {} \; | sort -u   # -> 0034

# 2. no Java 9+ API leaked in (incl. generated code)
javap -c -p -cp core/target/classes <fqcn> | grep -E "makeConcat|StringConcatFactory"            # -> empty

# 3. the OSGi execution environment bnd derived
unzip -p core/target/*.jar META-INF/MANIFEST.MF | tr -d '\r\n ' | grep -o 'osgi.ee=JavaSE)(version=[^)]*'
#   -> osgi.ee=JavaSE)(version=1.8
```

Class file versions: 52 = Java 8, 55 = 11, 61 = 17, 65 = 21.

**Also check embedded jars.** `Private-Package`/`-includeresource` content is not compiled by you, so a vendored client at class file 55 breaks on JVM 8 even though every class you wrote is correct. `unzip -l <bundle>.jar '*.jar'` lists them; run check 1 over each.

## Why this is safe for the other platforms

Class file 52 loads on every JVM from 8 upward, so one artifact still serves JVM 8 (classic 6.5), 11, 17 and 21 (6.5 LTS, AEMaaCS). The baseline is strictly more conservative, not a fork. OSGi agrees: a JavaSE 11 framework publishes `osgi.ee` capabilities for 1.8 as well, so a bundle requiring `JavaSE 1.8` resolves there.

The forward direction is not *unconditionally* safe, though — it is safe for the **class file format**, while APIs removed from later JDKs still bite: JAXB (`javax.xml.bind.*`), JAX-WS and CORBA went in Java 11; `Thread.stop()` became a hard throw in 20; `SecurityManager` was removed in 24; and Java 16+ denies reflective access to JDK internals. Relevant to a **translation** connector: **CLDR became the default locale data provider in Java 9**, changing locale display names and date/number formatting — so derive locale labels from an explicit mapping table, never from JDK display names, or output varies with the customer's JVM.

## AEM-specific note

Classic **AEM 6.5 supports both Java 8 and Java 11**, so the lowest common baseline for a single-artifact connector is a *customer* question, not a product one — ask which JVM their 6.5 actually runs on before assuming 11. (6.5 LTS requires 17/21 and AEMaaCS is 11+, so only classic 6.5 can force an 8 baseline.) This is the floor; [aem-sdk-java-ceiling-dual-65-cloud.md](aem-sdk-java-ceiling-dual-65-cloud.md) is the corresponding ceiling on the build JDK.

Verified on the Phrase AEM connector (aem-sdk-api 2026.8.27830 + uber-jar 6.5.22, build JDK 21, Lombok 1.18.46), Sept 2026.