# WKND on AEM 6.5 on-prem: picking the package and the "where is the content?" gotcha

## Two flavors per release

The WKND reference site (`adobe/aem-guides-wknd` on GitHub) ships every release
as two zips on the [releases page](https://github.com/adobe/aem-guides-wknd/releases):

- `aem-guides-wknd.all-<v>.zip` — AEM as a Cloud Service
- `aem-guides-wknd.all-<v>-classic.zip` — **AEM 6.5 on-prem** (use this one)

There is also a small `aem-guides-wknd.ui.content.sample-<v>.zip` per release —
that's tutorial sample content only, not the full site.

## Version ↔ AEM 6.5 compatibility (from the repo README, Aug 2026)

| WKND | AEM 6.5 | Java (build) |
|---|---|---|
| 4.x (latest 4.0.4) | 6.5 **LTS** (+ current AEMaaCS) | 21 (main branch) |
| 3.2.0 | below 6.5.23 (≈ up to SP22) | 8, 11 |
| 1.1.0 | 6.5.10+ | 8, 11 |

Rule of thumb: instance on Java 11 / regular service pack → `3.2.0-classic`;
6.5 LTS → `4.0.4-classic`. The 4.x line dropped the Java 8/11 toolchain.

## The content IS in the package — it just installs later

Verified by unzipping `aem-guides-wknd.all-3.2.0-classic.zip` (135 MB): it is a
container package whose sub-packages sit under
`jcr_root/apps/wknd-packages/**/install/` and `wknd-vendor-packages/**/install/`:

- `aem-guides-wknd-shared.ui.content-2.2.2.zip` — **138 MB, the full site
  content + DAM assets** (nearly the whole download)
- `aem-guides-wknd.ui.content-3.2.0.zip` + `ui.content.sample-3.2.0.zip` —
  site structure and sample pages
- `ui.apps`, `ui.config`, `core` jar, Core Components 2.20.8 (+ config/content)

**Gotcha:** those sub-packages are picked up *asynchronously* by the JCR/OSGi
installer after Package Manager reports the container "installed". The 138 MB
content package takes several minutes on its own, so right after install
`/content/wknd` genuinely does not exist yet — this looks like "the package has
no content" but is just lag.

Verify / troubleshoot, in order:

1. Wait a few minutes; check `/sites.html/content/wknd` and
   `/assets.html/content/dam/wknd-shared`.
2. `/crx/packmgr` — `aem-guides-wknd-shared.ui.content` appears there once the
   embedded install ran; it can also be installed manually from there.
3. `/system/console/slinginstaller` — OSGi installer status, shows stuck or
   failed embedded installs.

No external dependencies needed — Core Components are embedded in the classic
package.

Confirmed by package inspection Aug 2026 (WKND 3.2.0-classic).