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

## The mutual-exclusivity trap

Adobe documents `/sessionmanagement` and `/allowAuthorized "1"` as **mutually exclusive**. They are contradictory orders: "cache only for validated sessions" vs "cache ignoring auth." In the exam scenario (CUG pages not being cached despite `/sessionmanagement` being added), the cause is a leftover `/allowAuthorized "1"` in `/cache` — the fix is flipping it to `"0"`, *not* touching `/rules`, `/clientheaders`, or `/stickyConnectionsFor`. The misread to avoid: "`1` should mean it caches *more*, so it can't be the blocker" — with session management present, the session gate dominates, requests without a valid session are proxied and **not** cached, and `"1"` prevents the gate from ever validating anyone. (Mechanism as documented + consistent reading; not reproduced against a live Dispatcher.)

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

Farm-level: the whole farm becomes a secured area. A session is created when a request carrying the configured `/header` is answered successfully by AEM; afterwards, cached content is served only to requests whose session validates.

**The real-world gotcha the exam skips:** AEM CUG login is form-based and yields a `login-token` **cookie**, not an `Authorization` **header** — so `/header "HTTP:authorization"` never sees CUG logins and no session is ever established. A production config for cookie-based login points `/header` at the cookie (`Cookie:login-token`). Exam answers stop at the `/allowAuthorized` flip; a working deployment needs both.

## /auth_checker (the AEMaaCS-era alternative)

Per-request validation instead of farm-level sessions: the Dispatcher sends a **HEAD** request for the page to a configurable AEM endpoint (a custom authorization servlet); 200 → serve from cache, anything else → deny/forward. Scoped by `/url`, `/filter` (which paths get checked) and `/headers` (which request headers are passed to the servlet). This is the pattern Adobe documents for permission-sensitive caching on AEMaaCS, where you author only `filters/`, `cache/`, `clientheaders/` etc. in the dispatcher module — same principle: works only with `/allowAuthorized "0"`.

## Related but distinct

- Responses carrying `Set-Cookie` are never cached regardless of these settings; `/filter` is request security, `/ignoreUrlParams` is cache-key hygiene ([dispatcher-ignoreurlparams.md](dispatcher-ignoreurlparams.md)); general cache/invalidation rules in [dispatcher-caching-and-invalidation.md](dispatcher-caching-and-invalidation.md).
- CUG itself (what's being protected) is an AEM-side ACL/authentication-requirement mechanism; the Dispatcher only decides whether its *cache* may answer for AEM.

Adobe dispatcher docs + exam material, Oct 2026; flag semantics documented, interaction behavior not lab-verified.
