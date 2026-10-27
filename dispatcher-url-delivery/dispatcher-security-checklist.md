# Dispatcher security checklist

Adobe's pre-production checklist for the Dispatcher, condensed into what to do and why. Source: the "Dispatcher Security Checklist" page of the Dispatcher guide. Docs-based. Adobe says to complete the AEM security checklist for your AEM version as well; this page covers only the Dispatcher side. Related: [[dispatcher-fundamentals]] (the `/filter` section), [[dispatcher-caching-and-invalidation]] (flush agents), [[cors-policy-and-dispatcher]].

## The checklist

| Item | What to do | Why |
|---|---|---|
| **Latest Dispatcher version** | Install the newest build for your web server. The running version is in `dispatcher.log`: `Dispatcher initialized (build 4.x.y)`. | Security fixes ship only in new builds. The module is versioned separately from AEM. |
| **Restrict who can flush the cache** | Limit flush requests to the Publish instances (or the flush agents) by IP, in Apache or in `/allowedClients` of the farm. | Anyone who can send a flush request can empty the cache and push load to Publish. |
| **HTTPS everywhere** | Enable TLS on Author and Publish, not only on the public edge. | Credentials and `login-token` cookies travel on the internal hops too. |
| **Restrict access with `/filter`** | Deny everything, then allow only the paths the site needs. | Default deny stops unreviewed URLs from reaching AEM. |
| **Block administrative URLs** | Make sure filters deny the Web Console, CRXDE, the OSGi console and the other admin paths. Adobe's "Testing Dispatcher Security" page lists the URLs. | These give repository or system access if reachable from outside. |
| **Allowlists, not blocklists** | Write filter rules as "deny all, allow these", never "allow all, deny these". | A blocklist misses every URL nobody thought of. |
| **Dedicated system user** | Run the web server as a least-privilege user with write access only to the cache folder. On IIS, set "Connect as a specific user" on the site's physical path. | Limits what a compromised web server process can touch. |
| **DoS protection** | Validate URLs with `mod_rewrite` where the patterns are simple. Limit caching to expected extensions (`.html`, `.jpg`, `.gif`, `.js`, `.pdf`, …) so junk URLs do not fill the cache. | Unlimited extensions let an attacker create endless cache files. |
| **`/etc` and `/libs`** | Deny `/etc/` and `/libs/`, then allow only the paths Publish needs: `/etc/designs/*`, `/etc/clientlibs/*`, `/etc/segmentation.segment.js`, `/libs/cq/personalization/*`, `/libs/cq/security/userinfo.json`, `/libs/granite/security/currentuser.json` (must not be cached), `/libs/cq/i18n/*`, `/libs/wcm/stats/tracker.js`, `/libs/cq/personalization/components/clickstreamcloud/content/config.json`. | Those two trees hold product code and configuration. Only a few files are needed by the browser. |
| **CSRF** | Allow `/libs/granite/csrf/token.json` in `/filter` and add `CSRF-Token` to `/clientheaders`. | AEM's CSRF framework needs the token endpoint reachable and the header forwarded, or POSTs from the site fail. |
| **Clickjacking** | Send `X-Frame-Options: SAMEORIGIN` from the web server. | Stops the site being framed by another origin. |
| **Penetration test** | Test the whole AEM infrastructure before go-live. | The checklist is a floor, not proof of security. |

## Notes

- The `/etc` and `/libs` allow list is from the AEM 6.x era. On a current project, client libraries live under `/apps` or `/etc.clientlibs`, and the personalization and stats paths only matter if those features are used. Allow what your site actually requests, which you can see in the Dispatcher log at debug level.
- "Restrict access" and "block admin URLs" are the same `/filter` section. The admin-URL item is the specific list to check after the deny-all rule is in place.
- `currentuser.json` and `userinfo.json` must be allowed but not cached. Put them in `/filter` as allow and in `/cache` → `/rules` as deny, or they return another user's data from cache.

## What this means for AEM

On AEM 6.5 and AMS every item is yours. On AEMaaCS, Adobe runs the Dispatcher module, the system user, TLS at the CDN and the Dispatcher version, so those items are Adobe's. What remains yours is the dispatcher module in the project: `/filter` rules (deny all, allow needed paths), cache rules by extension, `clientheaders` including `CSRF-Token`, and response headers such as `X-Frame-Options` in the Apache `vhost` files. The Cloud Manager pipeline validates the dispatcher configuration, and the Dispatcher SDK runs the same checks locally.
