# Dispatcher `/ignoreUrlParams`

Controls which URL **query-string parameters** the Dispatcher treats as cache-relevant. Solves the classic "campaign tracking params cause cache misses and overload Publish" scenario.

---

## Plain-English version

Think of Dispatcher as a photocopier sitting in front of a slow printer (AEM). Normally it just hands out a copy it already made instead of bugging the printer.

Ads and links often stick junk on the end of a URL, like `?utm_source=facebook` vs `?utm_source=instagram` — a tracking sticky note that doesn't change the page at all. The copier is dumb by default: a different sticky note looks like a totally different page, so it refuses to reuse old copies and runs to the printer every time. Enough ad clicks with enough different sticky notes and the printer melts.

`ignoreUrlParams` is telling the copier "ignore that sticky note, it doesn't change the page — hand out the copy you already have." But some junk *does* matter, like `?search=shoes` — for those you say "no, don't ignore this one, go get a fresh copy from the printer."

## What it does

By default, Dispatcher treats every distinct query string as a **distinct URL** for caching purposes — no normalization. `?utm_source=a` and `?utm_source=b` on the same page are different cache entries (or, depending on config, a query string present at all can disqualify the request from being served from cache and send it straight to Publish/AEM).

`/ignoreUrlParams` (a section under `<farm>/cache/`) tells Dispatcher which query params to **ignore when deciding cache hit/miss** — i.e. which params do *not* affect the rendered page content and so shouldn't fragment the cache.

## Config shape

Structurally identical to `/filter`: an ordered glob allow/deny list, **last match wins**.

```
/cache {
  ...
  /ignoreUrlParams {
    /0001 { /glob "*"       /type "0" }   # baseline: every param is cache-relevant (deny-all)
    /0002 { /glob "utm_*"   /type "1" }   # exception: ignore campaign tracking params
    /0003 { /glob "gclid"   /type "1" }
    /0004 { /glob "fbclid"  /type "1" }
  }
}
```

- `/type "0"` = **don't** ignore this param (it's cache-relevant; a request with it is treated as its own URL / may bypass cache).
- `/type "1"` = **ignore** this param (strip it for cache-key/hit purposes).
- Two equally valid patterns, pick based on which list is shorter:
  - **Allowlist-the-ignorable** (deny-all baseline, `1` for known-safe tracking params) — safer default, but only params you've explicitly listed get cache benefit; forget to add a new tracking param and it silently starts bypassing cache.
  - **Wildcard-ignore-all, deny-the-dynamic** (`/glob "*" /type "1"` baseline, then `/type "0"` for params that genuinely change content, e.g. search/`q=`) — used when campaign/tracking params vastly outnumber the few functionally-significant ones. This is the pattern the exam scenario (many campaign params, one search param) wants: ignore everything by default, explicitly un-ignore only the handful of dynamic params.

## Where it sits relative to other Dispatcher sections

| Section | Controls |
|---|---|
| `/filter` | Request **allow/deny at the security layer** — is this request permitted to reach AEM at all. Nothing to do with caching. |
| `/ignoreUrlParams` | Which **query params matter for the cache key / hit-or-miss decision**. |
| `/rules`, `/statfileslevel` | Which **paths** are cacheable at all, and invalidation depth. |

These are independent knobs — allowing a param through `/filter` does nothing for caching, and vice versa. A common exam trap is offering a `/filter` change as the "fix" for a caching performance problem (see question 48, options A/D) — `/filter` is security, not cache optimization.

## Exam signal

Symptom pattern: performance degrades specifically under **campaign/tracking traffic** with **unique query params per hit**, while a *separate* query param genuinely drives dynamic content (search, personalization). The fix is a Dispatcher **config change** (`/ignoreUrlParams`), not app-level URL rework (rewriting params to selectors) — that's a valid alternative but heavier than the problem requires when a config-only fix exists.

## References
- Question 48 in this repo (`questions/question-48.md`) — the beauty-company campaign-traffic scenario this note was written from.
- Related: [[exposing-apis-to-external-systems]] — Dispatcher/Publish-tier boundary reasoning for external-facing endpoints generally.
