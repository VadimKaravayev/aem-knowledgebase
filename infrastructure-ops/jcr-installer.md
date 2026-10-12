# JCR Installer: the repository-to-OSGi bridge

The JCR Installer Provider (part of the Apache Sling installer framework,
status page at `/system/console/osgi-installer`, a.k.a. `slinginstaller`) scans
the repository and hands what it finds to the OSGi installer. It is the
mechanism behind "put a config file in an install folder and it becomes an OSGi
configuration" — and behind why that configuration then *stays* what the
deployed code says it is.

**Scope:** Sling *JCR Installer Provider* docs (sling.apache.org) and Adobe
*OSGi Configuration Settings* (AEM 6.5), both read October 2026. Behavior not
reproduced against a running instance except where a linked note says
otherwise.

Related:
[osgi-config-runmode-resolution.md](osgi-config-runmode-resolution.md) (which
folder wins when several match),
[wknd-onprem-install.md](wknd-onprem-install.md) (the async-install gotcha in
the wild),
[tarmk-cold-standby.md](tarmk-cold-standby.md) (the write-back persist flag as
a standby-killing trap).

---

## What it scans

- **Search paths with weights:** `/apps` and `/libs` by default; artifacts
  found under `/apps` get higher priority, which is why an `/apps` config masks
  a `/libs` one.
- **Folder name regexp:** in plain Sling only folders named `install` are
  scanned; AEM ships a regexp that also covers `config` folders (the
  "Installation folders name regexp" and "Max hierarchy depth" parameters on
  the *Apache Sling JCR Installer* OSGi config — rarely worth changing).
- **Run-mode suffixes:** `install.<runmode>` / `config.<runmode>` folders are
  scanned only when *all* their run modes are active. Priority is **base 50
  plus 1 per matching run mode**, so `config.emea.author` (52) beats
  `config.author` (51) beats `config` (50) — the numeric form of the
  winner-takes-all-per-PID rule in
  [osgi-config-runmode-resolution.md](osgi-config-runmode-resolution.md).

## What it installs

| Found in the repo | Becomes |
|---|---|
| `.cfg.json` / `.config` file, or a `sling:OsgiConfig` node | OSGi configuration in ConfigAdmin |
| Bundle jar in an install folder | Installed + started OSGi bundle |
| Content package (zip) in an install folder | Handed to Package Manager for installation |

Bundle **start level** comes from a numeric parent folder:
`/apps/x/install/15/bundle.jar` starts at level 15.

## The two properties that matter operationally

1. **Deployed state is authoritative.** A configuration the installer applied
   is marked in the Web Console as *bound to a configuration resource*. Edit it
   in the console and the next (re)install of the owning package reasserts the
   file's values. This is the exam-grade answer to "apply an OSGi config that
   cannot be changed at runtime": a config file in an install/config folder,
   not a console edit, not system properties.
2. **It is asynchronous.** The installer drains its own queue *after* Package
   Manager reports "installed". Embedded sub-packages and bundles can land
   minutes later — an apparently empty content tree right after a package
   install is queue lag, not a broken package
   ([wknd-onprem-install.md](wknd-onprem-install.md)). Check
   `/system/console/slinginstaller` before re-installing anything.

## Write-back (6.5/AMS; the drift machine)

Console edits travel the *other* direction too: with
`org.apache.sling.installer.configuration.persist=true` (the default), a config
changed in the Web Console is written back into the repository as a `.config`
file — new configs under `/apps/sling/install`, edits to a `/libs`-delivered
config as a copy under `/apps`. That is where the `/apps/system/config` drift
described in
[osgi-config-runmode-resolution.md](osgi-config-runmode-resolution.md) comes
from, and leaving persist enabled on a TarMK cold standby makes it sync the
primary's config down and stop being a standby
([tarmk-cold-standby.md](tarmk-cold-standby.md), trap 1). On AEMaaCS the Web
Console doesn't exist, so neither does this path.

## Pausing

Since installer 3.1.8 the scan can be paused by creating a child node under
`/system/sling/installer/jcr/pauseInstallation` — for operations where a
half-installed artifact mid-scan would hurt (e.g. large content imports).
Remove the node to resume; a forgotten pause node means "nothing installs and
nothing errors".

## Reading

- Apache Sling, *JCR Installer Provider* —
  <https://sling.apache.org/documentation/bundles/jcr-installer-provider.html>
- Adobe, *OSGi Configuration Settings* (AEM 6.5) —
  <https://experienceleague.adobe.com/en/docs/experience-manager-65/content/implementing/deploying/configuring/osgi-configuration-settings>
