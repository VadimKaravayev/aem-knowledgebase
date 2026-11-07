# Dispatcher and authenticated content: /allowAuthorized, /sessionmanagement, /auth_checker

How the Dispatcher treats requests from logged-in visitors (CUG/protected content), and the setting interaction that question-writers love because almost everyone misreads it.

## /allowAuthorized is not "cache on/off" — it's "may the security check be skipped?"

The flag answers one question: **may the Dispatcher serve/store cached copies for authenticated requests without any permission check?**

- `"1"` = yes, skip the check — protected pages are treated like public ones (cached once, served to *anyone* who requests the URL)
- `"0"` (default) = no, never skip the check

What "authenticated request" means to the Dispatcher: it carries an `Authorization` header, a cookie named `authorization`, or a cookie named `login-token`.

## The three postures (pick exactly one per farm)

1. **`/allowAuthorized "0"`, nothing else** — the Dispatcher has no way to check visitors itself, and "never skip the check" therefore means: forward every authenticated request to AEM, cache nothing for them. Safe, zero cache benefit for logged-in users. This is where the intuition "`0` = don't cache" comes from — true *only in this posture*.
2. **`/allowAuthorized "1"`** — cache and serve ignoring auth entirely. Only safe when all logged-in users see identical content **and** nothing in the farm is actually permission-restricted (or auth is enforced upstream).
3. **Permission-sensitive caching** — `/sessionmanagement` (farm-level secure sessions) or `/auth_checker` (per-page HEAD check against an AEM servlet): **cache the page once, check each visitor at the door** before serving the cached copy. Cache-hit performance with per-user security. Requires posture 1's flag: `/allowAuthorized "0"`.

The doorman picture that makes posture 3 click: `0` = "always check the membership card"; `/sessionmanagement`/`/auth_checker` = hiring the doorman who *can* check cards. Together: copies are handed out fast, but only to verified members. `1` = "don't check anyone" — which makes the doorman unemployable.

## The one-line rule, verbatim from Adobe

The Dispatcher configuration reference, section "Enable secure sessions - /sessionmanagement", carries this caution:

> `/allowAuthorized` Set to `"0"` in the `/cache` section to enable this feature. As detailed in the section "Caching when authentication is used" section, when you set `/allowAuthorized 0` requests that include authentication information are not cached. If permission-sensitive caching is required, see the Caching Secured Content page.

That is the whole rule: **to use `/sessionmanagement`, set `/allowAuthorized "0"`.** Two things the caution adds that the exam answer does not: with `"0"` and no session management, authenticated requests are simply not cached; and Adobe points to the *Caching Secured Content* page (`/auth_checker`, below) as the feature it calls "permission-sensitive caching", so `/sessionmanagement` and `/auth_checker` are two distinct features that both require `"0"`. Adobe does not use the phrase "mutually exclusive"; that wording comes from exam material.

## The trap

`/sessionmanagement` and `/allowAuthorized "1"` are contradictory orders: "cache only for validated sessions" vs "cache ignoring auth." In the exam scenario (CUG pages not being cached despite `/sessionmanagement` being added), the cause is a leftover `/allowAuthorized "1"` in `/cache` — the fix is flipping it to `"0"`, *not* touching `/rules`, `/clientheaders`, or `/stickyConnectionsFor`. The misread to avoid: "`1` should mean it caches *more*, so it can't be the blocker" — with session management present, the session gate dominates, requests without a valid session are proxied and **not** cached, and `"1"` prevents the gate from ever validating anyone. (Mechanism as documented + consistent reading; not reproduced against a live Dispatcher.)

Distractor taxonomy from the same question, reusable: `/rules` = *which paths* are cacheable (can't authorize anything); `/clientheaders` = header forwarding (already `"*"` in most dumps — an "add header X" answer that's covered by a wildcard changes nothing); `/stickyConnectionsFor` = load-balancing pin to one render, takes a *content* path (an option feeding it a filesystem path is a category error).

## /sessionmanagement mechanics (what the sub-keys do)

```
/sessionmanagement {
  /directory "/usr/local/apache/.sessions"   # session store on the dispatcher host (must exist, httpd-writable)
  /encode "md5"                              # how the session key is hashed
  /header "HTTP:authorization"               # WHERE the dispatcher looks for the credential
  /timeout "800"                             # seconds of session validity
}
```

Farm-level: the whole farm becomes a secured area. Adobe's description is that a session is created when a request carrying the configured `/header` is answered by AEM; afterwards, cached content is served only to requests whose session validates. Requests without the header, or with a header value that has no session, are forwarded to AEM uncached (the CUG then returns 401 or the login redirect), so anonymous visitors never see the cached copy. That part works as advertised.

### Lab-verified: the session is created regardless of AEM's answer

Reproduced on the AEMaaCS SDK Dispatcher in Docker in front of an SDK publish, with a CUG page, `/allowAuthorized "0"` and `/sessionmanagement { /header "Cookie:login-token" /encode "md5" /timeout "120" }`, after the page had been cached once by a real login:

```
# a cookie value that was never issued by AEM
curl -s -o /dev/null -w '%{http_code}\n' -b 'login-token=never-logged-in' http://localhost:8080/content/site/en/private.html   # 401  (forwarded, publish rejects)
curl -s -o /dev/null -w '%{http_code}\n' -b 'login-token=never-logged-in' http://localhost:8080/content/site/en/private.html   # 200  (dispatcher.log: hit)
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080/content/site/en/private.html                                   # 401  (no header, forwarded)
```

The first request is proxied and refused by publish, but the Dispatcher writes a session file under `/directory` anyway (one file per distinct header value, named by its `/encode` hash). From the second request on, for `/timeout` seconds, that cookie value unlocks every cached page in the farm. The Dispatcher only checks "does a session exist for this header value", never "did AEM accept it". Consequences:

- Any client can read every cached protected page by sending `login-token=<anything>` twice. Session management is a cache-hit optimisation for farms where *all* content is secured and the credential header is trustworthy on its own, not a security gate.
- AEM-side logout does not end the Dispatcher session; it lives until `/timeout`.
- This is the practical reason Adobe's AEMaaCS guidance is `/auth_checker`, which asks publish about the specific page on every cache hit.

Related browser-side gotcha seen in the same lab: the cached response's header sidecar (`page.html.h`) only replays the headers listed in `/cache/headers`, and a CUG page rendered by AEM carries no `Cache-Control`, so Chrome happily re-displays a protected page from its *own* cache after logout. Checking Dispatcher behaviour needs "Disable cache" in DevTools or curl, and a real deployment should send `Cache-Control: private` (or `no-store`) on protected pages.

**The real-world gotcha the exam skips:** AEM CUG login is form-based and yields a `login-token` **cookie**, not an `Authorization` **header** — so `/header "HTTP:authorization"` never sees CUG logins and no session is ever established. A production config for cookie-based login points `/header` at the cookie (`Cookie:login-token`). Exam answers stop at the `/allowAuthorized` flip; a working deployment needs both.

## /auth_checker (the AEMaaCS-era alternative)

Per-request validation instead of farm-level sessions. The Dispatcher ships an **AuthChecker module**; when `/auth_checker` is configured it sends a **HEAD** request for the page to a servlet on Publish and serves the cached copy only on 200. Scoped by `/url` (servlet path), `/filter` (which paths get checked; `allow` = "check this path", `deny` = "skip the check", not "block") and `/headers` (which headers of the checker's *response* are passed back to the client, typically `Set-Cookie:*`). Works only with `/allowAuthorized "0"`. This is the pattern Adobe documents for permission-sensitive caching on AEMaaCS.

### Two servlets, not one — the naming trap

Adobe's "Caching Secured Content" page names two distinct things and states the distinction outright: *"The render calls the AEM authorizer servlet (this servlet is not the Dispatcher AuthChecker servlet)."*

| | Auth Checker servlet | AEM authorizer |
|---|---|---|
| Who provides it | **You write and deploy it** on Publish (`sling.servlet.paths` = the `/url` value, e.g. `/bin/permissioncheck`) | **Built into AEM** — the normal authentication + ACL check that runs when a page is rendered |
| Who calls it | The Dispatcher's AuthChecker module, via HEAD, when a cached copy exists | Sling, whenever a request is actually forwarded to the render |
| What it guards | The **cache** — may this visitor receive the cached file? | **Rendering** — may this visitor get the page from AEM at all? |
| Output | HTTP status only: 200 = serve from cache, 403 = deny | The rendered page, or a login redirect / 403 |

The three documented flows:

1. **Cached, authorized** — Dispatcher → HEAD to Auth Checker servlet → 200 → cached file served. AEM authorizer never runs.
2. **Not cached, authorized** — Dispatcher forwards to the render → AEM authorizer passes the user → page rendered → Dispatcher caches it.
3. **Not authorized** — Auth Checker servlet returns non-200 → Dispatcher forwards to the render → AEM authorizer decides what the user sees (login page or 403). Nothing cached.

Both gates must agree: the Auth Checker servlet is only as good as the permission check it performs, and the AEM authorizer alone cannot protect files already in the cache.

### Auth Checker servlet requirements (from the docs)

- Implement `doHead` only — the Dispatcher sends HEAD requests exclusively.
- Must be **accessible to all users** (not behind a CUG, not blocked by `/filter`), otherwise the check itself fails and nothing is served.
- Read the requested path from the `uri` request parameter, check it with the visitor's session (`session.checkPermission(uri, Session.ACTION_READ)` in Adobe's sample), return 200 on success and 403 on failure.
- Keep it cheap: it runs on every request to a protected path, including cache hits. Resolve/check only, never render.
- The servlet path must be allowed in the Sling Servlet Resolver's execution paths.

### Lab-verified: /auth_checker next to /sessionmanagement

Reproduced on the AEMaaCS SDK Dispatcher in Docker in front of an SDK publish. Farm: `/allowAuthorized "0"`, the `/sessionmanagement` block from above (`/header "Cookie:login-token"`), plus:

```
/auth_checker {
    /url "/bin/authcheck"
    /filter {
        /0000 { /glob "*" /type "deny" }
        /0001 { /glob "/content/site/en/secure/*" /type "allow" }
    }
    /headers {
        /0000 { /glob "*" /type "deny" }
        /0001 { /glob "Set-Cookie:*" /type "allow" }
    }
}
```

The servlet answered 200 for members of one group and 403 otherwise (any rule works, the Dispatcher only reads the status). Observed by comparing publish's `request.log` with the servlet's own log lines:

| Request | What reached publish | Served from |
|---|---|---|
| Member (`login-token` from a real login), page already cached | `HEAD /bin/authcheck` only → 200 | cache |
| Fake `login-token`, sent twice (session file exists after the first) | `HEAD /bin/authcheck` → 403, **then `GET page.html`** | publish |
| No `login-token` at all | `GET page.html` only, no HEAD | publish |

What this settles:

- **A deny is not a 403 to the browser.** On a non-200 from the checker the Dispatcher refuses only the *cached copy* and forwards the original request to publish, returning whatever publish answers (flow 3 above). With no CUG/ACL on the page, publish answers 200 and the deny is invisible from the browser; it shows only in `request.log` as the extra GET. **So the CUG on publish stays mandatory**; the checker protects the cache, never the page.
- **The fake-cookie gap is closed for the checked paths.** The session file is still created for the fake value (session management behaves as in the section above), but the checker vetoes the cache hit. Paths outside the checker's `/filter` keep the gap.
- **With `/sessionmanagement` on, a request without the session header never reaches the checker.** The debug log shows `Session manager: no authorization information found` → `cache-action ...: NONE`: the cache is skipped and the request is forwarded, so only publish's own access rules apply.
- **Startup log wording differs from the docs.** Adobe's page quotes `AuthChecker: initialized with URL 'configured_url'.`; this Dispatcher build (4.3.8) logs `Authorization checker: initialized with URL '/bin/authcheck'.` at debug level.
- **Never delete the docroot folder itself in the SDK.** Emptying the bind-mounted cache including its `html/` folder makes Apache answer 403 for every URL (`AH01630: client denied by server configuration: /mnt/var/www/html`) until the folder is recreated or the container restarted; the container creates it only at startup. Flush with `rm -rf cache/html/*`.

## Related but distinct

- Responses carrying `Set-Cookie` are never cached regardless of these settings; `/filter` is request security, `/ignoreUrlParams` is cache-key hygiene ([dispatcher-ignoreurlparams.md](dispatcher-ignoreurlparams.md)); general cache/invalidation rules in [dispatcher-caching-and-invalidation.md](dispatcher-caching-and-invalidation.md).
- CUG itself (what's being protected) is an AEM-side ACL/authentication-requirement mechanism; the Dispatcher only decides whether its *cache* may answer for AEM.

Adobe dispatcher docs + exam material; the `/sessionmanagement` "session created even on 401" behaviour and the browser-cache gotcha are lab-verified on the AEMaaCS SDK Dispatcher (see the lab-verified sections), as are the `/auth_checker` cache-hit and deny flows next to `/sessionmanagement`; the remaining `/auth_checker` flows are docs-based.
