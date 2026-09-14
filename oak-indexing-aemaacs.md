# Oak Indexing in AEM as a Cloud Service

How search indexing works in AEMaaCS, how it differs from AEM 6.5, and the rules/gotchas for shipping custom index definitions. Source: [Adobe — Content Search and Indexing](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/operations/indexing).

---

## Background: what an Oak index is

An index is a precomputed lookup structure so queries don't have to **traverse** every node (O(N) full scan — the source of the *"query without index / traversed 10000+ nodes"* log warnings). You pay extra work on **writes** to keep the index organized so **reads** are cheap.

- **Property index** — a sorted `value → node-paths` map. Fast exact-match lookups (O(log N)). In Oak these are **synchronous**: updated inside the same commit as the content change, so always accurate. Cost is paid on the write path.
- **Lucene (full-text) index** — an **inverted index** mapping `term → documents` (tokenize text once at index time; queries become a direct term lookup + list intersection). Expensive to build (tokenizing, segment merges), so Oak runs it **asynchronously**.

### Synchronous vs asynchronous (the core gotcha)

- **Sync (property):** index updated in the same transaction → always correct, slower writes.
- **Async (Lucene):** content commits immediately; a background `AsyncIndexUpdate` job catches the index up on a timer (~every 5s). Between the write and the next run there is a **lag window** where new content isn't searchable yet, deleted content still appears, and edited values show stale. This async lag is the classic cause of "search returns inaccurate data."

On AEM **6.5**, one fix for accuracy-sensitive lookups was to switch to a **synchronous property index**. **On AEMaaCS that escape hatch does not exist for custom indexes** (see below).

---

## AEMaaCS specifics

### Only Lucene is supported for custom indexes
> "Index management on AEM as a Cloud Service is only supported for indexes of type `lucene`."

- ❌ No custom **property** indexes (the 6.5 sync-property fix is unavailable).
- ❌ No external **Solr** (removed).
- A single Lucene index definition can still carry property `indexRules` for exact-match constraints — but those are evaluated **asynchronously** like the rest of the Lucene index.

**Nuance:** under the hood AEMaaCS queries may be served by **Elasticsearch**, not raw Lucene. You still author Oak/Lucene-format definitions; the platform translates. Implementation detail, invisible to authors.

### `/oak:index` is immutable → config-as-code
You **cannot** edit indexes on a running instance (no Felix/JMX/CRXDE editing of `/oak:index` like 6.5). All index definitions ship via **code deployment** (package under `/apps`).

### Naming conventions
- **Extend an OOTB index:** `<indexName>-<productVersion>-custom-<customVersion>` — e.g. `damAssetLucene-6-custom-1`
- **Brand-new custom index:** `<prefix>.<indexName>-<productVersion>-custom-<customVersion>`, where `prefix` is a **2–5 char** project id.
- The `-custom-N` token marks the definition as replacing/extending a base one. **Bump N** on each change to trigger a fresh version.

### Blue-green reindex on deploy (zero downtime)
On deployment AEM keeps **two index versions side-by-side** (old + new), **reindexes the repo against the new definitions before switching traffic**, then drops the old. Queries never hit a half-built index. The platform **auto-merges** your customizations with Adobe's updated OOTB definitions.

---

## Where the index data physically lives (and the node paths)

The indexer reads the repo once at build time and stores, per matching node, its **JCR path** plus the indexed property values — into a Lucene structure. That data exists in **two copies** (verified on a local 6.5/SDK quickstart with `bdk.dcFormatLucene-1-custom-1`):

### Layer 1 — authoritative, inside the repository
A **hidden child node** of the index definition:
```
/oak:index/bdk.dcFormatLucene-1-custom-1/
    :data     ← Lucene segments as binaries in the NodeStore (segmentstore .tar / blobstore)
    :status
```
`:data` is **colon-prefixed = hidden** — Oak does NOT expose it as a JCR node, so it never shows in CRXDE or `.json` output (a `.tidy.N.json` of the index node stops at `indexRules`). This is the source of truth.

### Layer 2 — local-disk working copy (what queries actually read)
Lucene needs fast random file access, so Oak's **`IndexCopier` (copy-on-read)** copies the segments out of the NodeStore onto local disk the first time the index is used. Queries run against these files:
```
crx-quickstart/repository/index/bdkdcFormatLucene1custom1-<timestamp>/
    index-details.txt        ← maps back: indexPath=/oak:index/bdk.dcFormatLucene-1-custom-1
    data/
        segments_1, segments.gen   ← commit point / generation (which segments are current)
        _0.si                      ← Segment Info (doc count, codec)
        _0.cfe                     ← Compound File Entries (offsets into .cfs)
        _0.cfs                     ← Compound File Segment: THE DATA — terms dict + postings + stored fields
```

**The node paths live in `_0.cfs`**, as Lucene **stored fields** — Oak keeps each doc's JCR path in an internal **`:path` field**. Query flow: term lookup (`dc:format:text/css`) → matching Lucene doc IDs → read their `:path` field → the result paths. That's literally where "which node is where" is recorded.

The `-<timestamp>` suffix on the folder is a version: on reindex (bump `-custom-N`), Oak copies a fresh version into a NEW directory and switches over, then deletes the old — the local-disk side of blue-green.

**Chain:** indexer → `:data` in NodeStore (truth) → IndexCopier → local `data/_0.cfs` (query copy) → `:path` stored field → result paths.

> AEMaaCS nuance: when queries are served by the **Elasticsearch** backend, the built data lives in Adobe-managed Elasticsearch instead of `:data`/local `.cfs` files — same definition node, different data home. The two-copy `:data` + IndexCopier model above is the Oak-Lucene (6.5 / local SDK) story.

---

## Restrictions vs AEM 6.5
- ❌ No direct Index Manager / `/oak:index` editing on a single instance.
- ❌ **No custom analyzers** — built-in analyzers only.
- ❌ **No vector similarity** search (`useInSimilarity = true` unsupported).

## Best practices / gotchas
- When customizing an OOTB index (e.g. `damAssetLucene`), **copy the base definition from a real Cloud Service environment, not the local SDK** — they can differ.
- **Keep the Tika configuration** when copying a definition (binary text extraction).
- **Keep total index size growth under ~100%** — excessive growth can **block the deployment**.

## Out-of-the-box indexes (managed, don't edit directly)
`lucene` (catch-all full text), `damAssetLucene`, `cqPageLucene`, `ntBaseLucene`/`nodeTypeLucene`, plus internal property/reference/counter indexes (`uuid`, `principalName`, `reference`, `nodetype`, `counter`). Customize by **extending** them, never editing.

## Tooling
- **Oak Indexing Tools (oakTools)** — guided UI to generate/manage custom index definitions: https://oak-indexing.github.io/oakTools/index.html
- **Index Manager UI** (diagnosis): `/libs/granite/operations/content/diagnosistools/indexManager.html`
- **Developer Console** per-environment for query/index diagnostics.

---

## How to create a custom index (step-by-step)

Creating an index is a **code deployment**, not an instance action — there is no "create index" button.

### 1. Extend an OOTB index, or create a new one?
Decides the node name:

| Case | Name pattern | Example |
|---|---|---|
| Extend OOTB | `<ootbName>-<productVersion>-custom-<n>` | `damAssetLucene-6-custom-1` |
| Brand new | `<prefix>.<name>-<productVersion>-custom-<n>` (prefix 2–5 chars) | `acme.productLucene-1-custom-1` |

When extending: **copy the base definition from a real Cloud Service environment, not the local SDK** (they differ), and keep the existing `tika` config.

### 2. Author the definition
Lives at `/oak:index/<yourIndexName>` in `/apps` (a `ui.apps`/`ui.content` index package), as `.content.xml`:

```xml
<!-- /oak:index/acme.productLucene-1-custom-1/.content.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0"
          xmlns:oak="http://jackrabbit.apache.org/oak/ns/1.0"
          xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="oak:QueryIndexDefinition"
    type="lucene"
    async="[async,nrt]"
    compatVersion="{Long}2"
    evaluatePathRestrictions="{Boolean}true">
  <indexRules jcr:primaryType="nt:unstructured">
    <acme:Product jcr:primaryType="nt:unstructured">   <!-- node type covered -->
      <properties jcr:primaryType="nt:unstructured">
        <sku jcr:primaryType="nt:unstructured"
             name="sku" propertyIndex="{Boolean}true"/>      <!-- exact match -->
        <title jcr:primaryType="nt:unstructured"
             name="title" analyzed="{Boolean}true"
             nodeScopeIndex="{Boolean}true"/>                 <!-- full text -->
      </properties>
    </acme:Product>
  </indexRules>
</jcr:root>
```

Key knobs: `type="lucene"` (only supported type); `async="[async,nrt]"` — `nrt` (near-real-time) shrinks indexing lag, the closest thing to the old sync-property freshness; `propertyIndex=true` for exact match; `analyzed`/`nodeScopeIndex` for full text. **No custom analyzers, no `useInSimilarity`.**

### 3. Validate before deploying
The Cloud Manager build pipeline runs **index definition validation** — bad syntax or **>100% index-size growth blocks the deploy**. Use **oakTools** to generate a correct definition + naming/version scaffold; validate against the local SDK first.

### 4. Deploy via Cloud Manager
Commit → run pipeline. Platform does the **blue-green reindex**: builds the new version in the background, reindexes the repo, switches query traffic only when fully built (zero downtime), then drops the old version.

### 5. Iterate by bumping `-custom-N`
Never edit in place. Change `...-custom-1` → `...-custom-2` and redeploy; the platform builds the new and retires the old. This is also the rollback path.

### 6. Verify it is used
- **Index Manager UI** — confirm it exists/built/refresh status.
- **Query explain** (Query Performance tooling) — confirm Oak's cost-based planner picks *your* index, not a traversal or a different one. Tune properties / `evaluatePathRestrictions` if not.

**One-liner:** author a `type=lucene` `oak:QueryIndexDefinition` at `/oak:index/<name>-<ver>-custom-N` in `/apps`, validate (size + syntax), deploy via Cloud Manager (blue-green reindex), bump `-custom-N` to change, verify with Index Manager + query-explain.

---

## Facets and the `secure` flag (ACL-check bypass)

Faceted search (e.g. Assets console facet counts) makes Oak do a **per-result ACL check** — for every candidate node, confirm the querying user can actually see it — before it's counted in a facet bucket. With many authoring groups and broad/complex permission structures, this per-result check becomes the bottleneck, even when every group effectively has read access to everything (i.e. the ACL check can never actually deny anyone — it's pure overhead).

Two things are required to facet on a property, and they live at different levels of the index definition:

1. A **`facets` config node** as a direct child of the index definition (sibling of `indexRules`), with a `secure` attribute:
   - `secure-simple` (default) — full per-result ACL check, always accurate, slowest.
   - `insecure` — **skips the ACL check entirely** for facet computation. Only safe when the facet content is effectively public to everyone who can run the query (as in the AEMaaCS "15+ authoring groups, all with read access to the whole repo" scenario) — otherwise this leaks facet counts (and thereby existence) of content a user shouldn't see.
   - `statistical` — approximates counts via sampling instead of checking every result; a middle ground when `insecure` isn't safe but `secure-simple` is too slow.
   - Optional `topChildren` caps how many facet values are returned per field.
2. **`facets="{Boolean}true"`** on each individual property's rule under `indexRules` — marks that property as facetable at all. `secure` on the `facets` node has no effect on a property that isn't marked this way.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0"
          xmlns:oak="http://jackrabbit.apache.org/oak/ns/1.0"
    jcr:primaryType="oak:QueryIndexDefinition"
    type="lucene"
    async="[async,nrt]"
    compatVersion="{Long}2">
  <facets jcr:primaryType="nt:unstructured"
      secure="insecure"
      topChildren="{Long}100"/>
  <indexRules jcr:primaryType="nt:unstructured">
    <dam:Asset jcr:primaryType="nt:unstructured">
      <properties jcr:primaryType="nt:unstructured">
        <myFacetProp jcr:primaryType="nt:unstructured"
            name="jcr:content/metadata/myProperty"
            propertyIndex="{Boolean}true"
            facets="{Boolean}true"/>
      </properties>
    </dam:Asset>
  </indexRules>
</jcr:root>
```

**Diagnosis tell:** slow facet computation, no traversal warnings in the logs (so the index itself is being used fine), and permission structure that's complex (many groups) but not actually restrictive (all groups can read everything) — that combination points at ACL-check overhead, not index design, and the fix is `secure="insecure"` on `facets`, not a leaner index or more indexed properties.

## Maven build / FileVault validation gotchas (the painful ones)

Shipping an index in a standard AEM Maven project (`filevault-package-maven-plugin`) trips **five** separate validators. Each emits a different error; all five must be satisfied. Learned the hard way deploying `bdk.dcFormatLucene-1-custom-1` to a `ui.apps` (packageType `application`) project.

### 1. The dot in the index name is a NODE NAME, not a path separator
Index name `bdk.dcFormatLucene-1-custom-1` must be **one** folder on disk:
```
ui.apps/src/main/content/jcr_root/_oak_index/bdk.dcFormatLucene-1-custom-1/.content.xml   ✅
```
NOT nested folders `_oak_index/bdk/dcFormatLucene-1-custom-1/` ❌ — that creates two nodes, the parent `bdk` defaults to `nt:folder`, and you get a cascade of "not contained in filter", "orphaned filter entry", and `nt:folder`-can't-hold-`oak:QueryIndexDefinition` errors. (`/oak:index` → `_oak_index` is the usual FileVault colon encoding; the dot is just a literal char in the folder name.)

### 2. `allowIndexDefinitions` is a TOP-LEVEL plugin parameter, NOT a validator option
The `jackrabbit-oakindex` validator (`OakIndexDefinitionValidator`) blocks index defs by default. The toggle is a **direct mojo parameter** of the filevault plugin, a **sibling of `validatorsSettings`** — putting it under `<validatorsSettings><jackrabbit-oakindex><options>` does NOTHING (validator still reports "allowIndexDefinitions is not set to 'true'"):
```xml
<configuration>
  <allowIndexDefinitions>true</allowIndexDefinitions>   <!-- HERE, top-level -->
  <validatorsSettings>...</validatorsSettings>
</configuration>
```

### 3. APPLICATION packages reject content outside `/apps` and `/libs`
`jackrabbit-packagetype` errors: *"Package of type 'APPLICATION' is not supposed to contain content outside root nodes 'apps' or 'libs'"*. Add `oak:index` to the allowed immutable roots:
```xml
<jackrabbit-packagetype>
  <options><immutableRootNodeNames>apps,libs,oak:index</immutableRootNodeNames></options>
</jackrabbit-packagetype>
```

### 4. `/oak:index` must be a valid filter root
`jackrabbit-filter` errors: *"Filter root's ancestor '/oak:index' is not covered … nor a valid root"*. Add it via the filter validator's `validRoots` option (note: this **replaces** the default valid-roots list, but `/apps` etc. stay valid because the **structure package dependency** still covers them — the extra `/content/dam` "not a valid root" lines drop to harmless WARNINGS):
```xml
<jackrabbit-filter>
  <options><validRoots>/oak:index</validRoots></options>
</jackrabbit-filter>
```

### 5. Child `validatorsSettings` does NOT reliably merge with the parent pom's
The AEM archetype parent pom sets `<validatorsSettings>` (with `jackrabbit-nodetypes` cnd) in `<pluginManagement>`. A child `<validatorsSettings>` does **not** merge at runtime the way `help:effective-pom` *displays* — the parent block wins and your settings vanish. Force it with `combine.self="override"` and **re-include the parent's nodetypes cnd** so nodetype validation still works:
```xml
<validatorsSettings combine.self="override">
  <jackrabbit-nodetypes><options><cnds>tccl:aem.cnd</cnds></options></jackrabbit-nodetypes>
  <jackrabbit-filter><options><validRoots>/oak:index</validRoots></options></jackrabbit-filter>
  <jackrabbit-packagetype><options><immutableRootNodeNames>apps,libs,oak:index</immutableRootNodeNames></options></jackrabbit-packagetype>
</validatorsSettings>
```

### 6. Do NOT add `/oak:index` to the structure package
Tempting fix for #4, but a bare `<filter root="/oak:index"/>` in `ui.apps.structure` means "this package owns the entire `/oak:index` subtree" → would wipe all OOTB indexes, and `allowIndexDefinitions` does **not** whitelist it (*"filter rule overwriting a potential index definition below '/oak:index'"*). Keep the structure package out of it; solve #4 with `validRoots` on `ui.apps` instead.

### Deploy + verify (local)
- Deploy just the index without the full reactor: `mvn -pl ui.apps -PautoInstallPackage clean install` (set `JAVA_HOME` to the required JDK — this project enforces Java 21+).
- Confirm node built: `GET /oak:index/<name>.tidy.5.json` → `reindex:false`, `reindexCount:1`, your `indexRules/.../properties/<prop>` present with `propertyIndex:true`.
- Confirm it's used: **Query Performance → Explain Query**. Plan flips from `/* traverse allNodes (warning: slow) estimatedEntries: 204900 */` to `/* lucene:<name> … luceneQuery: dc:format:text/css estimatedEntries: 282 */`.

---

## References
- [Adobe — Content Search and Indexing (AEMaaCS)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/operations/indexing) — the Cloud Service rules (Lucene-only custom indexes, naming, blue-green, restrictions).
- [Apache Jackrabbit Oak — Indexing](https://jackrabbit.apache.org/oak/docs/query/indexing.html) — canonical engine reference: synchronous / asynchronous / NRT modes, index definitions, **indexing lanes**, **checkpoints**, the **diff index** customization feature, corrupt-index isolation, cluster considerations, and **when to reindex vs avoid it**.