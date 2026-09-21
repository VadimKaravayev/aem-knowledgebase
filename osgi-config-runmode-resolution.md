# OSGi config resolution: run-mode folders replace, they do not merge

The rule that costs people a day: when two `config.<runmode>` folders both match
a running instance, the more specific one **replaces the other for the whole
PID**. It is not a per-property override. Every property you did not restate in
the winning folder falls back to the bundle's compiled-in default — not to the
value sitting in the less specific folder you assumed was still the base layer.

Nothing warns you. The config simply behaves as if half of it were never
written, and only on instances carrying the extra run mode.

**Scope:** sourced from Adobe, *Configuring OSGi* (AEM **6.5**), read September
2026. The resolution algorithm carries to AEMaaCS; the Web Console editing path
does not. Version differences are called out per section. Not reproduced
against a running instance.

Related: [all-package-embed-structure.md](all-package-embed-structure.md)
(where `ui.config` puts these folders),
[repoinit-acls-on-apps-and-libs.md](repoinit-acls-on-apps-and-libs.md) (the
repoinit factory config that lives in one),
[rolling-deployment-two-version-overlap.md](rolling-deployment-two-version-overlap.md)
(the closed AEMaaCS run-mode set).

---

## The resolution algorithm

Folders live under your project and are named by run mode:

```
/apps/<project>/config                    # every run mode
/apps/<project>/config.author             # one run mode
/apps/<project>/config.emea.author        # two — order in the name is irrelevant
```

Two rules, applied in order:

1. **A folder is a candidate only if *all* of its run modes are active at
   startup.** `config.author.dev` on an `author,dev,emea` instance: candidate.
   `config.author.asean` on the same instance: not a candidate, because `asean`
   is not active. Partial matches score nothing; they are excluded outright.

2. **Among candidates for the same PID, the highest number of matching run
   modes wins.** On `author,dev,emea`:

   | Folder | Matches | Outcome |
   |---|---|---|
   | `config` | 0 | loses |
   | `config.author` | 1 | loses |
   | `config.emea.author` | 2 | **applied** |

### The part that bites

> *"You cannot define some properties for the same PID in
> `/apps/*/config.author/` and more specific ones in
> `/apps/*/config.emea.author/` for the same PID. The configuration with the
> highest number of matching run modes is effective for the entire PID."*

Winner-takes-all, per PID. Now combine it with the other documented rule —

> *"You must only create properties for the parameters that you want to
> configure, others still take the default values as set by AEM"*

— and the failure mode is fully formed:

```
config.author/            com.example.MyService
    endpoint = https://internal.example.com
    timeout  = 30000
    retries  = 5

config.emea.author/       com.example.MyService
    endpoint = https://emea.example.com
```

On an EMEA author instance the service runs with the EMEA endpoint and
`timeout`/`retries` **at the bundle's `@AttributeDefinition` defaults**. Not
30000 and 5. The `config.author` node is not a base layer; it is a competitor
that lost.

**Therefore: every run-mode-specific folder must carry the complete
configuration for its PID.** Duplicate the shared properties into each variant,
or keep the PID in exactly one folder and drive the difference some other way
(a property placeholder, a separate PID, or a service that reads the run mode
itself).

A useful sanity check when a config "partly works": count the run modes on the
instance, list every folder that could match, and ask which one won — then look
at what that single folder actually contains.

### On AEMaaCS

Same algorithm, smaller input space: run modes are the closed set `author` /
`publish` crossed with `dev` / `stage` / `prod` / `rde`, combined as
`<service>.<environment>` — so the realistic collision is `config.publish`
versus `config.publish.prod`, and it behaves exactly as above. See
[all-package-embed-structure.md](all-package-embed-structure.md) for where
these folders sit inside `ui.config` (`/apps/<app>/osgiconfig/config.*`), and
note the separate, stricter rule that **package embed** targets accept only
`install.author` / `install.publish` — config folders are the one place
environment scoping is available at all.

---

## Two precedence chains, and they are not the same

**At startup**, lowest wins last:

1. `/apps/*/config...` — your project
2. `/libs/*/config...` — product defaults
3. `.config` files in `<cq-install>/crx-quickstart/launchpad/config/...`

A generic `/libs` configuration is **masked** by a project-specific one in
`/apps`. This is why the standing instruction is to copy a `/libs` config into
`/apps` before customising rather than editing it in place — the masking is the
supported mechanism, and editing `/libs` gets overwritten by the next service
pack.

Product configs really do ship per run mode, e.g.:

```
/libs/wcm/core/config.author/com.day.cq.wcm.core.WCMRequestFilter
/libs/wcm/core/config.publish/com.day.cq.wcm.core.WCMRequestFilter
/libs/wcm/core/config.publish/com.day.cq.wcm.core.stats.PageViewStatistics
```

**At runtime**, the order differs — the Web Console jumps to the top:

1. Web Console modifications (immediate)
2. `/apps` modifications (immediate)
3. `/libs` modifications (immediate unless masked by `/apps`)

---

## The Web Console is run-mode-blind, and writes outside your project

**6.5 / AMS only.** Two properties of `/system/console/configMgr` that together
produce most "works on this box, nowhere else" drift:

- Changes are *"applied immediately and applicable to the current instance,
  **irrespective of the current run mode**."* The console does not consult the
  folder-resolution rules at all — there is no run mode to match, because you
  are writing directly to ConfigAdmin.
- The change persists **into `/apps`, but not into your project**. Default
  landing spot is `/apps/system/config`. A config originally read from
  `/libs/foo/config/someconfig` is written back to the mirrored path
  `/apps/foo/config/someconfig`.

So a console tweak on a production author becomes a node under
`/apps/system/config` that no `ui.apps` filter covers, no build produces, and
no code review sees — and it outranks the repository config at runtime. It
survives until someone goes looking.

The console is genuinely the best way to *author* a config — it knows the
parameter names, types and allowed values, and the page recommends making the
change there rather than hand-writing `.config` syntax. The discipline is to
treat the console output as a **source to copy into `ui.config`**, then remove
it, rather than as the configuration itself.

On **AEMaaCS** this route is closed: there is no writable OSGi console, and all
configuration ships as code in `ui.config`. The drift class described here
cannot occur — which is most of the reason Adobe closed it.

---

## Never edit `/crx-quickstart/launchpad/config`

That directory is OSGi Configuration Admin's **private persistence store** —
`*.config` files holding every configuration regardless of how it was entered.
Adobe's instruction is explicit: never edit the folders or files under it.

Worth knowing for diagnosis rather than editing: it is why deleting a
repository config node does not always restore the default. ConfigAdmin already
holds the value, and the repository node was only the delivery mechanism. The
fix is to change the value through a supported path, not to reach into the
store.

---

## Node naming: PID, factory PID, and the two separators

A standard configuration is a node (type `sling:OsgiConfig`) or a `.config`
file named exactly for the PID:

```
com.day.cq.wcm.core.impl.VersionManagerImpl
```

A **factory** configuration appends an identifier of your choosing — free text
whose only job is to distinguish instances of the same factory:

```
org.apache.sling.commons.log.LogManager.factory.config-MINE      # hyphen (classic)
org.apache.sling.jcr.repoinit.RepositoryInitializer~author       # tilde (newer)
```

Both separators are in live use and belong to different eras: `-` is the
convention documented for `sling:OsgiConfig` nodes and `.config` files in 6.5;
`~` is the OSGi R7 / Sling form you see with `.cfg.json`, and is what the
repoinit examples in
[repoinit-acls-on-apps-and-libs.md](repoinit-acls-on-apps-and-libs.md) use.
Pick the one matching the file format you are writing rather than assuming they
are interchangeable — that interchangeability is not something this doc has
verified on 6.5.

Property names are the ones shown in brackets in the console's field
descriptions, e.g. `[versionmanager.createVersionOnActivation]`. The PID itself
appears in brackets after the bundle name on the Configuration tab.

---

## Finding where a config actually lives

The audit recipe for the drift above. CRXDE Lite → Tools → Query, SQL:

```sql
select * from nt:base where jcr:path like '/apps/%'
  and contains(*, 'org.apache.felix.webconsole.internal.servlet.OsgiManager')
```

→ `/apps/system/config/org.apache.felix.webconsole.internal.servlet.OsgiManager.config`

And to inventory everything node-based on the instance:

```sql
select * from sling:OsgiConfig
```

Run the second one against a long-lived author and compare the result to what
your `ui.config` produces; the difference is your accumulated console drift.
Note this finds `sling:OsgiConfig` **nodes** only — configs delivered as
`.config` or `.cfg.json` files are `nt:file` and need the first query's path/
content search instead.

CRXDE Lite is unavailable on a production-ready 6.5 instance
([production-ready-mode-crxde.md](production-ready-mode-crxde.md)) and on
AEMaaCS ([aemaacs-repository-inspection-by-tier.md](aemaacs-repository-inspection-by-tier.md));
on the latter the Developer Console's read-only Repository Browser is the
equivalent.

---

## References

- [Configuring OSGi (AEM 6.5, Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/implementing/deploying/configuring/configuring-osgi)
- [all-package-embed-structure.md](all-package-embed-structure.md)
- [repoinit-acls-on-apps-and-libs.md](repoinit-acls-on-apps-and-libs.md)
- [rolling-deployment-two-version-overlap.md](rolling-deployment-two-version-overlap.md)
