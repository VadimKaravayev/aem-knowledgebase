# CORS in AEM: the Granite policy, the dispatcher's two halves, and default-deny

Browser-side cross-origin access to AEM is configured in **three places that
must agree** — a Granite OSGi policy, the dispatcher's inbound header list, and
the dispatcher's cached-header list. Getting two of the three right produces a
clean HTTP 200 with no CORS headers, which reads like a routing bug and is not
one.

**Sourcing:** Adobe, *Understand Cross-Origin Resource Sharing (CORS)* (AEM
Learn), read September 2026. Not reproduced against a live instance.
Inference is flagged where it appears.

Related: [exposing-apis-to-external-systems.md](exposing-apis-to-external-systems.md)
(which tier should answer at all),
[content-services-vs-cf-headless-delivery.md](content-services-vs-cf-headless-delivery.md)
(the endpoints these policies are usually written for).

---

## The policy is an OSGi factory

**PID:** `com.adobe.granite.cors.impl.CORSPolicyImpl`
**Console:** *Adobe Granite Cross Origin Resource Sharing Policy*

Each policy is **one factory instance**. You are expected to have several.

| Property | Type | Meaning |
|---|---|---|
| `alloworigin` | String[] | Exact origin URIs permitted |
| `alloworiginregexp` | String[] | Origins matched by regex |
| `allowedpaths` | String[] | **Regexes** for the resource paths this policy covers |
| `supportedmethods` | String[] | `GET`, `HEAD`, `POST`, `PUT`, `DELETE`, … |
| `supportedheaders` | String[] | Request headers the caller may send |
| `exposedheaders` | String[] | → `Access-Control-Expose-Headers` |
| `maxage` | Integer (s) | Preflight cache lifetime |
| `supportscredentials` | Boolean | Allows credentialed (`withCredentials`) requests |

It ships like any other OSGi config, in `ui.config` — which means the run-mode
folder rules apply, including the one where the most specific matching folder
replaces the others **for the whole PID** rather than merging. A policy split
across `config.publish` and `config.publish.prod` will lose half its properties
on prod. See
[osgi-config-runmode-resolution.md](osgi-config-runmode-resolution.md) and
[all-package-embed-structure.md](all-package-embed-structure.md).

---

## Selection: first match wins, and the default is deny

A request is matched on **two axes together** — its `Origin` header against
`alloworigin`/`alloworiginregexp`, and its path against `allowedpaths`. Then:

- **The first matching policy is applied.** Not the most specific, not a merge
  of all matches — the first.
- **No match → denied.**
- **No policies configured at all → denied.**

The consequence worth internalising: a broad early policy **shadows** a
narrower later one for the same origin. If you add a second policy to grant
`POST` to an origin that an existing read-only policy already matches, the new
policy never fires and the symptom is "my new CORS config does nothing" — an
ordering problem wearing the costume of a broken config. Adobe's own advice
points the same way: prefer **one policy per specific origin hostname** over
broad regexes, because origins have independent lifecycles and security needs.

### `allowedpaths` are regexes — escape accordingly

```
/content/site1/.*
/graphql/execute.json.*
/content/_cq_graphql/site1/endpoint.json
```

The third line is from Adobe's own example and the `.` characters are
**unescaped**, so each matches any character. Harmless there; not harmless in a
pattern where the over-match reaches a path you did not intend to expose.
Escape the dots you mean literally (`endpoint\.json`), anchor where you can,
and treat every value in this list as the regex it actually is. Nothing
validates this and an over-broad pattern fails **open**.

---

## The authenticated recipe, and the part everyone misses

Anonymous, read-only:

```json
{
  "supportscredentials": false,
  "supportedmethods": ["GET", "HEAD"],
  "alloworigin": ["https://site1.com", "http://127.0.0.1:3000"],
  "alloworiginregexp": ["https://.*\\.site1\\.com", "http://localhost:.*"],
  "allowedpaths": ["/content/site1/.*", "/graphql/execute.json.*"],
  "maxage:Integer": 1800,
  "supportedheaders": [
    "Origin", "Accept", "X-Requested-With", "Content-Type",
    "Access-Control-Request-Method", "Access-Control-Request-Headers"
  ]
}
```

Authenticated, with mutations — three additions, one of which is easy to miss:

```json
{
  "supportscredentials": true,
  "supportedmethods": ["GET", "HEAD", "POST", "PUT", "DELETE"],
  "allowedpaths": [
    "/content/site2/.*",
    "/libs/granite/csrf/token.json"
  ],
  "supportedheaders": [
    "Origin", "Accept", "X-Requested-With", "Content-Type",
    "Access-Control-Request-Method", "Access-Control-Request-Headers",
    "Authorization", "CSRF-Token"
  ]
}
```

1. `supportscredentials: true`.
2. `Authorization` and `CSRF-Token` added to `supportedheaders`.
3. **`/libs/granite/csrf/token.json` added to `allowedpaths`.**

The third is the one that bites. The client has to *fetch* a CSRF token
cross-origin before it can send a mutating request, so that endpoint needs its
own CORS coverage. Omit it and the mutation fails at the token fetch — a
failure that points at CSRF, at auth, at anything but the CORS path list.

**`alloworigin: ["*"]` is not a production configuration.** Adobe's wording:
absolutely not recommended, since it lets every foreign (attacker) site make
requests.

---

## The dispatcher: two edits, two sections, both required

**Inbound** — the headers must survive the trip to AEM. In `dispatcher.any`:

```
/clientheaders {
   ...
   "Origin"
   "Access-Control-Request-Method"
   "Access-Control-Request-Headers"
}
```

Without these the CORS handler never sees an `Origin` and cannot match a policy.

**Outbound** — the response headers must survive caching (Dispatcher 4.1.1+):

```
/publishfarm {
    /cache {
        /headers {
            "Access-Control-Allow-Origin"
            "Access-Control-Expose-Headers"
            "Access-Control-Max-Age"
            "Access-Control-Allow-Credentials"
            "Access-Control-Allow-Methods"
            "Access-Control-Allow-Headers"
        }
    }
}
```

Then **restart the web server** and **flush the cache entirely**. Cached
responses were stored with whatever headers were correct at the time; a policy
change does not retroactively fix them, and a half-flushed cache serves a mix.

### What is cacheable

| Tier | Auth | Cacheable |
|---|---|---|
| Publish | anonymous | **Yes** — headers cache alongside the content |
| Publish | authenticated | No — responses are user-specific |
| Author | any | Not practically |

### Why multi-origin on Publish is the hard case

Adobe scopes this OSGi configuration to **single-origin sharing on Publish**,
plus CORS access to Author, and sends multi-origin Publish setups to
dispatcher-specific documentation without saying why.

*(Inference, not stated on the page:)* a cached response carries exactly one
`Access-Control-Allow-Origin` value. Cache it for origin A and origin B gets
A's header; vary on `Origin` and you multiply the cache by the number of
origins. Caching and multi-origin are in direct tension, which is why the
answer moves up to the dispatcher/CDN layer rather than staying in an OSGi
policy. Treat that as the reasoning to verify, not as documented fact.

---

## Debugging

**The signature to recognise:** the request returns **HTTP 200 but the response
carries no `Access-Control-Allow-Origin`**. That is a CORS *denial*, not a
routing failure — AEM answered, the policy did not match, and the browser
discards the response. Stop inspecting routes and read the logs.

- Logger `com.adobe.granite.cors` at **DEBUG** prints the reason a request was
  denied; **TRACE** logs every request through the handler.
- **Replay with `curl`**, reproducing the XHR's headers exactly — particularly
  `Origin`. A `curl` without `Origin` will happily succeed and prove nothing.
- **Check what runs before CORS.** Authentication, CSRF and dispatcher filters
  can reject the request before the CORS handler is ever consulted; that looks
  identical from the browser.
- After any OSGi or `dispatcher.any` change: restart the web server, flush the
  cache, and only then retest.

---

## References

- [Understand CORS (AEM Learn, Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-learn/foundation/security/understand-cross-origin-resource-sharing)
- [exposing-apis-to-external-systems.md](exposing-apis-to-external-systems.md)
- [content-services-vs-cf-headless-delivery.md](content-services-vs-cf-headless-delivery.md)
- [osgi-config-runmode-resolution.md](osgi-config-runmode-resolution.md)
