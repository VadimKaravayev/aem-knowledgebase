# Sling Referrer Filter: what it checks, how to configure it, and its traps

Docs-based (Adobe's AEMaaCS headless "Referrer Filter" page, plus the CSRF section of the AEM 6.5 security checklist). Statements marked *unverified* are not confirmed against Sling source.

## What it does

`org.apache.sling.security.impl.ReferrerFilter` (Apache Sling Referrer Filter) is one of AEM's two CSRF defences. It inspects the `Referer` header of state-changing requests and rejects the request if the referring host is not on its allow list. The other defence is the Granite CSRF Filter (token-based); the two are independent, and a request can be rejected by either.

- Only methods in `filter.methods` are checked. GET is never checked, so a GET must never change state.
- It runs before your servlet / form handler, so a rejected request never reaches application code — the symptom is "my POST does nothing / 403" with nothing in the application's own logging.
- It is not about cross-site *pages*, only about the `Referer` value. A same-domain form still fails if the host the browser used is not allowed, or if the browser sends no `Referer` (e.g. a strict `Referrer-Policy`) while `allow.empty` is `false`.

## Configuration

File name: **`org.apache.sling.security.impl.ReferrerFilter.cfg.json`** (in `ui.config`, under the run-mode folder you need). Adobe's example:

```json
{
  "allow.empty": false,
  "allow.hosts": [ "my.domain" ],
  "allow.hosts.regexp": [ "" ],
  "filter.methods": [ "POST", "PUT", "DELETE", "COPY", "MOVE" ],
  "exclude.agents.regexp": [ "" ]
}
```

| Property | Meaning |
|---|---|
| `allow.empty` | Accept requests with no `Referer` header. Keep `false`. |
| `allow.hosts` | Exact trusted host names. |
| `allow.hosts.regexp` | Trusted hosts as regular expressions. |
| `filter.methods` | Methods that are checked. GET is not in the list. |
| `exclude.agents.regexp` | User-agent patterns that skip the check entirely (non-browser clients). |

The `[""]` entries in Adobe's example are presumably ignored as blank patterns (*unverified*). An empty regex would otherwise match every host / every user agent and silently disable the filter, so prefer `[]` in your own config.

## Traps

- **Not a factory config — one active configuration per service.** Your file **replaces** AEM's native Referrer Filter configuration rather than merging with it, and Adobe warns this may break product functionality. Copy the product defaults into your file and add to them, and re-check after product updates, which can change the native config you have overwritten.
- **Never use `*` as a host.** Adobe states that on AEMaaCS this disables authenticated GraphQL access and exposes the endpoint to everyone. Allow only specific trusted domains.
- **Headless GraphQL:** a Referrer Filter entry is needed for apps calling the GraphQL endpoint with **POST**. Persisted queries over **GET** do not need it. This is another reason to prefer persisted queries.
- **GraphQL schema exposure:** schemas are generated from enabled Content Fragment Models and are readable through the endpoint, so CF model field names should not carry sensitive information.

## Diagnosing a blocked POST on author

A plain HTML form POST that works on publish but fails on author has two prime suspects:

1. **Referrer Filter:** author reached through a hostname that is not in `allow.hosts` (internal name, load balancer, IP), or a missing `Referer` with `allow.empty=false`. Fix by adding the host, not by setting `allow.empty=true` or `*`.
2. **Granite CSRF Filter:** the logged-in author session requires a `:cq_csrf_token` / `CSRF-Token`, which only AEM's JavaScript adds (`granite.csrf.standalone` clientlib). Anonymous publish users are not checked, which is why the same form works there.

Ruled-out distractors: the HTML Library Manager, the WCM Debug Filter, and request-parameter handling cannot reject a request outright.

## What this means for AEM

- **6.5 / AMS:** configure through OSGi (Web Console or `ui.config`); listed in the 6.5 security checklist under "Protect against Cross-Site Request Forgery".
- **AEMaaCS:** `ui.config` only; the single-config / replace-not-merge rule and the "no `*`" warning matter most here because of headless GraphQL.

Related: [[cors-policy-and-dispatcher]], [[aemaacs-headless-cors]], [[dispatcher-security-checklist]].

Sources: Adobe Experience League, "Referrer Filter" (AEMaaCS headless deployment) and "AEM 6.5 Security Checklist".
