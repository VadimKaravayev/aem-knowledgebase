# The AEM as a Cloud Service SDK: artifacts, versioning, refresh routine

The SDK is what a developer runs and compiles against locally when the real
target is AEMaaCS. It is **not** identical to the cloud runtime; that gap is
why Rapid Development Environments exist. This note lists the artifacts,
where each comes from, how the version must track production, the monthly
refresh routine, and the CryptoSupport key trick that makes encrypted configs
survive a fresh quickstart.

Companion notes: [[aem-sdk-java-ceiling-dual-65-cloud]] (what happens when a
newer `aem-sdk-api` drags in a Java 17+ ceiling), [[java-target-platform-vs-build-jdk]],
[[cloud-manager-pipeline-steps]] (the same build/analyse steps run in the
cloud), [[production-ready-mode-crxde]] (run modes on a local quickstart).

---

## 1. The artifacts and where to get them

| Artifact | What it is | Source |
|---|---|---|
| **Quickstart Jar** | the AEM runtime for local development (author or publish by run mode) | Software Distribution Portal, zip |
| **Java API Jar** (`aem-sdk-api`) | every Java API you are *allowed* to compile against; the successor of the 6.5 **Uberjar** | Maven Central, `provided` scope |
| **Javadoc Jar** | docs for the API jar | Maven |
| **Dispatcher Tools** | local dispatcher validation and Docker image, separate UNIX and Windows builds | Software Distribution Portal, same zip |
| **6.5 Deprecated Java API Jar + Javadoc** | interfaces removed since 6.5, for migrating customers whose code no longer compiles | gated: ask Customer Support, needs a backend entitlement change |

- Software Distribution listings are visible only to people whose org has AMS
  or AEMaaCS environments.
- The API jar is the compile-time contract. If a class is not in
  `aem-sdk-api`, it is not a supported API on the cloud, even if it exists in
  the quickstart at runtime.

```xml
<dependency>
  <groupId>com.adobe.aem</groupId>
  <artifactId>aem-sdk-api</artifactId>
  <version>2026.8.27830</version>   <!-- match production; see §2 -->
  <scope>provided</scope>
</dependency>
```

## 2. Version discipline

- **The SDK version must match the AEM version running in production.** Find
  it in AEM under the `?` menu → About Adobe Experience Manager, or in Cloud
  Manager's environment card. Version strings look like
  `2026.8.27830.20260815T…Z-250700`; the leading `YYYY.M` is the release.
- **Refresh cadence** Adobe recommends: immediately after every **monthly
  maintenance release** (rebuild and retest), optional after **daily**
  releases, and in practice **biweekly**. Dispose of the full local state
  **daily** so the application never silently depends on stateful data.
- **Java**: build the SDK with a JDK distribution and version that Cloud
  Manager's build environment supports. The Adobe page still cites Oracle JDK
  11 extended support "until September 2026"; that window has effectively
  closed, and recent SDKs need Java 17+ anyway (details in
  [[aem-sdk-java-ceiling-dual-65-cloud]]).

## 3. Building for the SDK = a local dry run of Cloud Manager

The archetype build performs the same four steps the pipeline does:

1. **Compile** source into content packages.
2. **Build artifacts.**
3. **Analyse bundles** with the Maven analyser plugin (missing dependencies,
   API misuse, package structure).
4. **Deploy artifacts** to the local quickstart.

Because step 3 is the same analyser Cloud Manager runs, a clean local build
catches most "Build Images" failures before a 30-minute pipeline does.

## 4. Refresh routine for a new SDK version

1. Commit anything useful, or export it into a **mutable content package**.
   Local-only test content stays **out of the pipeline build**; keep it in a
   separate package under source control and reinstall it each refresh.
2. Stop the quickstart; move the old `crx-quickstart` folder aside.
3. Read the new AEM version from Cloud Manager, download the matching
   Quickstart Jar.
4. Start it in a **brand new folder** with the right run modes (rename the jar
   or pass `-r`). No remnants of the old instance.
5. Build the application, deploy via Package Manager, then install the
   mutable/test content packages.

## 5. CryptoSupport: make encrypted values portable between quickstarts

Encrypted OSGi properties (cloud service configs, SMTP passwords, anything via
the CryptoSupport API) are secured by a key **auto-generated at first start**.
In the cloud each environment reuses its own key automatically; locally, a
fresh quickstart means a fresh key and every encrypted value becomes garbage.

Fix, once per team:

1. Always start the local jar with
   `-Dcom.adobe.granite.crypto.file.disable=true`, which stores the key
   material in the repository at **`/etc/key`** instead of the data folder.
2. On the very first instance, build a content package with a filter on
   `/etc/key`. That package now holds the shared secret.
3. Export the mutable content that carries encrypted values (or copy them
   from `/crx/de`) into a package that is reused across installations.
4. On every fresh instance, install the `/etc/key` package first, then the
   config packages. Encrypted values decrypt because the key is in sync.

Treat the `/etc/key` package like a credential: keep it out of the
public repo and never let it reach a Cloud Manager pipeline.

## Exam checklist

- SDK = Quickstart Jar + `aem-sdk-api` (ex-Uberjar) + Javadoc + Dispatcher
  Tools; deprecated 6.5 API jar only via Support.
- Quickstart and Dispatcher Tools come from **Software Distribution**;
  the API jar comes from **Maven**.
- SDK version **must match production AEM**; refresh after every **monthly**
  release, optionally after daily ones.
- Local build runs the **same analyser** as Cloud Manager, so it is the
  cheapest place to catch deployment failures.
- SDK ≠ cloud runtime; for fast iteration against the real thing use an
  **RDE**.
- Local test content lives in a **separate package** so it never ships.
- Encrypted configs across fresh instances: `-Dcom.adobe.granite.crypto.file.disable=true`
  + a `/etc/key` package.

## References
- [The AEM as a Cloud Service SDK (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/developing/aem-as-a-cloud-service-sdk)
- [Rapid Development Environments (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/developing/rde/overview)
- [AEM Project Archetype (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-core-components/using/developing/archetype/overview)
