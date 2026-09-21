# Production-ready mode (`nosamplecontent`) and re-enabling CRXDE Lite

## The gotcha

Starting an AEM 6.5 / 6.5 LTS quickstart with the `nosamplecontent` runmode does two things, not one:

1. Skips installing the We.Retail sample content (the obvious, intended effect).
2. Switches the instance into **production-ready mode** — a bundle of security-hardening defaults (per the Adobe "Running AEM in Production Ready Mode" doc).

The second effect is what surprises you on a local dev instance: CRXDE Lite appears broken. Opening `http://localhost:4502/crx/de` loads the UI but shows:

> *If this is a production environment, CRX DE could be disabled for security reasons. If this is the case, you can choose to re-enable it by following the steps described here.*

## Why it looks half-broken

CRXDE Lite is two pieces:

- The **UI** at `/crx/de` (bundle `com.adobe.granite.crxde-lite`) — still Active and serves 200.
- The **backend**: the DavEx servlet serving the repository over HTTP at `/crx/server` (bundle `org.apache.sling.jcr.davex`).

In production-ready mode the DavEx servlet component (`org.apache.sling.jcr.davex.impl.servlets.SlingDavExServlet`) is simply left with **no OSGi configuration**, so the component never activates (`/system/console/components` shows state `no config`) and `/crx/server` returns 404. The UI detects the missing backend and shows the warning. Checking bundle states is a red herring — both bundles are Active.

Diagnostic signature:

```
curl -s -o /dev/null -w "%{http_code}" -u admin:admin http://localhost:4502/crx/de/index.jsp   # 200
curl -s -o /dev/null -w "%{http_code}" -u admin:admin \
  "http://localhost:4502/crx/server/crx.default/jcr%3aroot/.1.json"                            # 404 → this is the problem
```

## The fix

Create the missing OSGi config. No restart needed — the component activates the moment the config exists, and the config persists in the repository across restarts (one-time fix).

Via the Felix console UI: `/system/console/configMgr` → `org.apache.sling.jcr.davex.impl.servlets.SlingDavExServlet` → set `alias = /crx/server`, save.

Via curl:

```
curl -u admin:admin \
  -F "apply=true" \
  -F "action=ajaxConfigManager" \
  -F "propertylist=alias,dav.create-absolute-uri" \
  -F "alias=/crx/server" \
  -F "dav.create-absolute-uri=true" \
  "http://localhost:4502/system/console/configMgr/org.apache.sling.jcr.davex.impl.servlets.SlingDavExServlet"
```

Verify: `/crx/server/crx.default/jcr%3aroot/.1.json` now returns 200 and CRXDE Lite works.

Curl gotcha: use `-F` (multipart) fields exactly as above with a **single** URL. A malformed call (e.g. stray `-d` params plus a duplicate positional URL) can return 302 "success-looking" redirects while persisting nothing — always verify the component state afterwards, not just the HTTP code.

## Don't try to fix it by removing the runmode

`nosamplecontent`/`samplecontent` (like `author`/`publish`) are **installation-time runmodes**, frozen at first startup. Removing `nosamplecontent` from the start command of an already-installed instance changes nothing; the only way out is deleting `crx-quickstart/` and reinstalling (which would then also install We.Retail). The right approach on a dev instance is to keep the runmode and toggle the individual hardened defaults you actually need (CRXDE above; WebDAV etc. are similar one-off OSGi/config fixes).

Confirmed on AEM 6.5 LTS local author (quickstart jar, Java 21), July 2026.

## The same rule on the AEMaaCS SDK quickstart

The cloud SDK quickstart freezes the same class of decision at first start, and encodes it in the
**jar filename** rather than a start argument: `aem-<tier>_<environment>-p<port>.jar`, e.g.
`aem-author-p4502.jar`, `aem-publish_stage-p4503.jar`. Omitting the environment means `dev`. You copy
the one downloaded quickstart jar into a directory per instance and rename it.

- **Tier is baked into `crx-quickstart` at first start.** Renaming the jar from author to publish
  afterwards does nothing; delete `crx-quickstart/` and start over, exactly as with `nosamplecontent`
  above.
- **`-r prerelease` applies on the first start only**
  (`java -jar aem-author-p4502.jar -r prerelease`). Adding it to a later start of an existing
  instance has no effect.
- **Replication agents exist only in the local quickstart.** Real AEMaaCS moves content with Sling
  Content Distribution and the Adobe Pipeline and has no replication agents at all, so local
  author→publish has to be wired by hand on the default publish agent (transport URI
  `http://localhost:4503/bin/receive?sling:authRequestLogin=1`, admin/admin, agent user id blank).
  Anything built against that agent is local-only scaffolding that will not exist in the cloud.

Worth knowing before investing in a local repository: updating the SDK jar (monthly, after the last
Thursday) means replacing the whole local environment and losing its content. Keep sample content in
a package in Git, or move it across with oak-upgrade `includepaths`.

From the Adobe "Local Development Environment Set-up / AEM Runtime" tutorial (Sept 2026), not
reproduced locally. Current SDKs need **JDK 21**, and the abort message names a "Java Specification
11 VM" whatever the wrong JVM actually is.
