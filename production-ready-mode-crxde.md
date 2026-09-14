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
