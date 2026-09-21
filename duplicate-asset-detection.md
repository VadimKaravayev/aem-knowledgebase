# Duplicate asset detection (two different services, two different property names)

AEM ships **two unrelated duplicate-detection mechanisms** with confusingly similar purposes. Picking the wrong PID/property silently does nothing — the OSGi config deploys fine, the unknown factory PID is simply never bound.

Verified by disassembling the bundles on two local instances:

| | AEM 6.5 (on-prem) | AEMaaCS SDK |
|---|---|---|
| `cq-dam-core` | 5.12.164 | 5.15.278 |
| `cq-dam-processor-nui` | 1.0.8 | 1.1.976 |
| `CreateAssetServlet` / `detect_duplicate` | ✅ | ✅ |
| `AssetDuplicationDetector` / `enabled` + `detectMetadataField` | ❌ absent | ✅ |

## 1. Classic upload path — `CreateAssetServlet` (6.5 **and** cloud)

PID `com.day.cq.dam.core.impl.servlet.CreateAssetServlet` — "Day CQ DAM Create Asset Servlet". Metatype declares exactly **one** property:

```xml
<AD id="detect_duplicate" type="Boolean" default="false"
    name="detect duplicate"
    description="configuration to enable duplicate detection of uploaded asset"/>
```

```json
{ "detect_duplicate": true }
```

There is **no** `detectMetadataField` here — the hash field is hardcoded. String constants in `CreateAssetServlet.class`: `detect_duplicate`, `dam:isDuplicateAsset`, `duplicates`, `Error in detecting duplicate.` — i.e. it flags the new asset with `dam:isDuplicateAsset` and returns a `duplicates` list in the upload response.

AEM Guides / XML Documentation (`fmdita` utils bundle) piggybacks on the *same* PID: `com.adobe.dxml.upload.detectduplicate.DetectDuplicateService` reads `detect_duplicate` off `com.day.cq.dam.core.impl.servlet.CreateAssetServlet` via ConfigurationAdmin, then runs a hardcoded XPath:

```
//element(*, dam:Asset)[(jcr:content/metadata/@dam:sha1 = '<hash>')]
```

## 2. Cloud asset-compute path — `AssetDuplicationDetector` (AEMaaCS only)

PID `com.adobe.cq.assetcompute.impl.assetprocessor.AssetDuplicationDetector` — "Adobe AEM Cloud Asset Duplication Detector", in `cq-dam-processor-nui` 1.1.x. **Absent from 6.5** (nui 1.0.8 has no such class), so this config is cloud-only.

```xml
<AD id="enabled" type="Boolean" default="false" name="Enabled"/>
<AD id="detectMetadataField" type="String" default="dam:sha1"
    name="Detect metadata field"/>
```

Deploy as an OSGi config in a run-mode `config` folder under **`/apps`** (immutable code, `ui.apps`/`ui.config`), not `/conf` — `/conf/global/settings/...` is mutable context-aware configuration and the OSGi installer does not read `.cfg.json` from there:

```
ui.apps/src/main/content/jcr_root/apps/<project>/osgiconfig/config.author/
  com.adobe.cq.assetcompute.impl.assetprocessor.AssetDuplicationDetector.cfg.json
```

```json
{
  "enabled": true,
  "detectMetadataField": "dam:sha1"
}
```

Behavior, from the disassembled class:

- Config is resolved through **`AssetsConfigurationsResolver`**, not plain ConfigAdmin (log strings: `Failed to get asset duplication detector enabled configuration from assetsConfigurationsResolver`). So the effective value can be resolved per Assets configuration, not purely globally.
- Detection is a **QueryBuilder** predicate set: `type=dam:Asset`, `path=/content/dam`, `property=jcr:content/metadata/<detectMetadataField>`, `p.limit=100`, `p.guessTotal`, plus `jcr:createdBy`.
- On a match it **sends a notification** to the uploading user (`Duplicated assets found`, `Send notification for duplication detect result: '{}', userId: {}`) — it does *not* block the upload.
- Disabled → `The cloud assets duplication detection service is disabled`.

## Why `dam:sha1` satisfies "regardless of filename"

`dam:sha1` (on `jcr:content/metadata`) is a hash of the asset **binary**, so identical bytes hash identically no matter what the file is called. It is unique per *distinct binary*, not per asset — which is precisely what makes it usable as a duplicate key.

The hash exists to make the comparison affordable, not possible: a byte-for-byte diff would require fetching every existing binary out of the datastore on every upload (N blob reads), whereas the hash is one indexed property lookup. Oak's blob store already content-addresses binaries the same way, so identical uploads physically share one blob regardless of this feature; the detector is the author-visible layer over the same idea.

**Limitation:** exact-binary only. A re-exported JPEG, a resize, or one changed EXIF field produces a different `dam:sha1` and will **not** be flagged. Users routinely expect "duplicate detection" to mean perceptual similarity; it does not.

## Exam trap

Certification questions ask this as "detect duplicates regardless of filename" with options mixing the two PIDs' properties. The discriminator is **not** that one property name is invented — `detect_duplicate` and `enabled`/`detectMetadataField` are *both* real. It is that they belong to **different PIDs**, and an option pairing `detect_duplicate` with `detectMetadataField` is internally incoherent. Widely circulated answer keys get this wrong and assert `detect_duplicate` is fictional.

Also note the location half of the trap: `/apps` run-mode `config` folder (OSGi, immutable) vs `/conf/global/settings/dam` (context-aware config, mutable) — only the former is read by the OSGi installer.
