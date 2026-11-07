# Static Replication Agent

The out-of-the-box **Static Agent** on an AEM 6.5 author renders activated content and writes it as plain files to the **server's filesystem**, not to a publish instance and not into the repository. Adobe's own description: *"Agent that stores a static representation of a node into the filesystem"* and *"a specialized agent and (probably) is not required for most instances."*

---

## How it works

Activating a page triggers every enabled agent, including this one. Instead of a transport to publish, the Static Agent uses the **Static Content Builder** as its serialisation, which:

1. Picks the first matching line on the agent's **Rules** tab for the activated path.
2. Builds the URL from that rule and **fetches it over HTTP from the author itself**. Host and port come from the OSGi config *Static Content Builder* (`/system/console/configMgr`), default `localhost:4502`. It is a real HTTP request, so the page is rendered exactly as a browser would see it.
3. Authenticates that request with a **login token minted for the Agent User Id**. This is the central gotcha, see below.
4. Writes the response body to `<Directory>/<path><suffix>` on the **operating system filesystem** of the machine running AEM. The directory is the *Directory* field on the Rules tab, default `/tmp`.
5. Removes the item from its queue immediately. An empty queue right after activation is the normal, successful state, not a sign that nothing happened.

There is **no transport handler**. The Transport tab URI is irrelevant for this agent; the error *"Unable to replicate content with no transport handler"* seen in some forum threads is not the thing to fix.

## Anatomy of a rule

One rule per line on the Rules tab:

```
<path glob><space><request template>
```

- **Path glob**: matched against the activated node path. `/content/site*` matches the prefix itself and everything below; `/etc/design/*` matches only children. First match wins; no match means the node is silently skipped.
- **Request template**: the URL the builder requests. `${path}` expands to the activated path. What is appended decides the rendering:
  - `${path}.html?wcmmode=preview` or `?wcmmode=disabled` renders a page as HTML without the authoring UI. Needed because the request is served by author, which would otherwise return the editing chrome.
  - bare `${path}` returns the raw node, which for a DAM asset or a client library file is the binary itself.

The defaults as shipped:

```
/content/we-retail* ${path}.html?wcmmode=preview
/content/dam* ${path}
/etc/design/* ${path}
```

Pages become `.html` files, assets and client libraries are copied verbatim, so the output directory is a self-contained static mirror of the site.

## Where the files are, and where they are not

- Files land on the **OS filesystem of the host running the AEM process**: by default `/tmp/content/<site>/.../<page>.html`. On a remote VM or container, that is the remote filesystem, so SSH or `docker exec` into it before concluding nothing was written.
- **The `/tmp` node visible in CRXDE Lite is unrelated.** The agent never writes into the JCR. Looking for the output there is a dead end.
- The Directory value is an absolute path; a relative value resolves against the AEM process working directory.
- Output on disk is a mirror of the content path, so `${path}.html` for `/content/site/en/page` becomes `<Directory>/content/site/en/page.html`.

## Agent User Id must be a real user (confirmed on 6.5)

Because the builder authenticates its internal HTTP request with a **login token**, the Agent User Id must be an account for which a token can be created: a regular, enabled user. Leaving the field empty (system user) or putting a **service user** there produces an empty token, and the author answers the builder's request with **500**:

```
*ERROR* ... org.apache.felix.http.jetty Exception while processing request to /content/.../page.html
java.lang.IllegalArgumentException: Invalid token ''
    at org.apache.jackrabbit.api.security.authentication.token.TokenCredentials.<init>
    at com.day.crx.security.token.impl.TokenAuthenticationHandler.createCredentials
```

Adobe staff on the community forums referenced an internal defect for the same symptom with a service user on 6.3. Fix: set a regular user with read on the paths the rules cover (`admin` to prove the setup, then a dedicated normal user; never leave `admin` on a hardened instance, per the Security Checklist).

This is the one replication agent where "Agent User Id needs only read on the replicated paths" is true but insufficient: the user must *also* be token-capable.

## Log messages and what each one means

| Agent log line | Meaning | Look at |
|---|---|---|
| nothing at all after activation | agent disabled (ships disabled) or *Ignore default* ticked on Triggers | Settings and Triggers tabs |
| `Creating content for path /content/...` then silence | builder fetched and wrote the file; success | the OS directory on the AEM host |
| `Nothing to replicate for /content/...` | the fetch returned nothing: no matching rule, wrong host/port in the Static Content Builder OSGi config, or the render failed | the agent log at `Debug` level, then `error.log` at the same timestamp |
| `Unable to retrieve content: Unable to fetch content from http://localhost:4502/...: 500` | the internal render threw; with `Invalid token ''` in `error.log` it is the Agent User Id problem above | `error.log` |
| `...: 503` | same family on 6.3, reported for service users | same fix |

Set the agent log level to Debug on the Settings tab to see the rendered URL, the HTTP status and which rule was applied.

## What this means for AEM

- 6.5 / AMS only. AEMaaCS has no replication agents at all; publication is Sling Content Distribution, see [[content-distribution-journal]]. Static export there is a different design (Edge Delivery, or a crawler against publish).
- Use cases: serve a read-only site straight from a web server without a publish tier, produce a static archive of a site at a point in time, or feed an external static host. For normal delivery the Default Agent plus dispatcher is the tool.
- Exam framing: the Static Agent is the stock example of an agent whose target is the filesystem rather than a transport URI. The read-only Agent User Id rule from the Replication doc applies to the packaging step of every agent; this one additionally renders over HTTP as that user.

## References
- [Replication, AEM 6.5 (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/implementing/deploying/configuring/replication): "Replication Agents - Out of the Box", Agent User Id and transport fields.
- [Security Checklist, AEM 6.5, "Configure replication and transport users"](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/security/security-checklist)
- [Troubleshooting Replication, AEM 6.5](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/implementing/deploying/configuring/troubleshoot-rep)
- Community threads on the static agent with the same symptoms: [AEM 6.1 static replication agent](https://experienceleaguecommunities.adobe.com/adobe-experience-manager-sites-8/aem6-1-static-replication-agent-14828), [AEM 6.3 static agent configuration](https://experienceleaguecommunities.adobe.com/adobe-experience-manager-sites-8/aem-6-3-static-agent-configuration-220450)
- The `Invalid token ''` cause and the filesystem-vs-CRXDE confusion were reproduced on AEM 6.5.
