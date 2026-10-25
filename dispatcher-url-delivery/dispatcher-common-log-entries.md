# Dispatcher log entries you will actually meet — what each means and the fix

Adobe's AMS tutorial page *Dispatcher Common Logs* (AEM 6.5 / AMS; read 2026-10-25) as a lookup table, plus the triage context it leaves out. Log mechanics (`DispatcherLogLevel`, bring-up checklist, debug headers) are in [[dispatcher-fundamentals]]; cache semantics in [[dispatcher-caching-and-invalidation]]. On **AEMaaCS** the same module writes the same lines, but you read them via Cloud Manager `download-logs` / `tail-log` for the `aemdispatcher` log, not `/var/log/httpd` ([[aio-cloudmanager-cli]]).

## Paths on AMS

| What | Path |
|---|---|
| Dispatcher log | `/var/log/httpd/dispatcher.log` |
| Filters | `/etc/httpd/conf/publish-filters.any` (older layout) / `conf.dispatcher.d/filters/` |
| Enabled farms | `/etc/httpd/conf.dispatcher.d/enabled_farms/` (e.g. `300_weretail_publish_farm.any`) |
| Cached vanity list | `/tmp/vanity_urls` |
| Doc root (example) | `/mnt/var/www/author/` |

**Filter rejects only appear at Info or Debug level** — a quiet log at level Warn proves nothing.

## The entries

| Line (prefix `<level> pid …`) | Meaning | Fix |
|---|---|---|
| `W … Allowing requests with globs is considered unsafe.` | Dispatcher ≥ 4.2.x discourages `/glob` filter rules (`/glob "* *.css *"`) | rewrite with `/url`, `/extension`, `/path`: `/0041 { /type "allow" /extension "css" }` |
| `D … Filter rejects: GET /libs/granite/core/content/login.html HTTP/1.1` | a filter denied the request. Rejects for `/system/console/`, `/libs/granite/…login` on publish are **intended** | if a legitimate asset is blocked (e.g. `.eot` under `/etc/designs/`), add a targeted allow: `/0011 { /type "allow" /method "GET" /extension 'eot' /path "/etc/designs/*" }` |
| `W … Unable to connect socket to 10.43.3.40:4502: Connection timed out` | **TCP never opened**: wrong render IP/port in `/renders`, firewall, or AEM down/not listening | verify render IP:port, firewall, AEM process health |
| `I … "GET /favicon.ico" 502 - 54034ms authorfarm/-` | **TCP opened, AEM too slow**: dispatcher hit its render timeout → 502/503 to the client | raise `/timeout` in the farm's `/renders` entry, or fix AEM's health (threads, GC, slow queries) |
| `I … Current cache hit ratio: 87.94 %` | share of requests served from cache vs rendered | target **≥ 80 %**; a low ratio with caching configured usually means **too-frequent or too-broad flushes** |
| `E … Unable to create parent directory …: Not a directory` | normally a fetch racing a cache clear; otherwise wrong permissions or **SELinux context** on the doc root | `ls -Z /var/www/` → expect `drwxr-xr-x` and `httpd_sys_content_t` |
| `W … Unable to fetch vanity URLs from …:4503/libs/granite/dispatcher/content/vanityUrls.html: remote server returned: HTTP/1.1 404 Not Found` | `/vanity_urls` enabled in the farm but the **vanity URL feature pack is not installed** on the renderer (or anonymous cannot read it) | install the feature pack, grant anonymous read; healthy = `Loaded 18 vanity URLs from file /tmp/vanity_urls` |
| `W … No farm matches host 'we-retail.com', selected last farm 'publishfarm'` | no farm's `/virtualhosts` matched → **silent fallback to the last loaded farm** | fix typos / missing domains in `/virtualhosts`; order farms so the fallback is the harmless one |
| `I … "GET /content/we-retail/us/en.html" - + 24034ms publishfarm/0` | trailing `farm/render` = who served it; `+` = rendered by AEM and **now cached**; a repeat should be a cache hit | none — this is the baseline line to read timings and farm selection from |

## Counting what the filters block

```
grep "Filter rejects\|\!" /var/log/httpd/dispatcher.log \
  | awk 'match($0, /\/.*\//, m){ print m0 }' | awk '{ print $1 }' | sort | uniq -c | sort -rn
```

Top of that list = the paths to decide on: intended (admin consoles) vs. a missing allow rule.

## Reading the two timeouts apart (the part people conflate)

- **"Unable to connect socket … Connection timed out"** — the network layer; AEM never saw the request. Config/infra problem.
- **`502`/`503` with a large `ms` value** — AEM saw it and did not answer in time. Capacity or application problem; raising the dispatcher timeout hides it, pulling a thread dump on AEM finds it ([[thread-dump-analysis]]).

## Traps

- Reading a Warn-level log and concluding "no filter rejects" — they are logged at Info/Debug.
- Fixing a reject with `/glob` — the next line in the log is the glob warning.
- "No farm matches host" never errors; the wrong site serves from the last farm until someone reads the log.
- A high hit ratio right after a full flush is impossible; measure after warm-up, and look at flush frequency before cache rules.
- The cache-directory error on an otherwise healthy box is almost always SELinux, not disk.

## References

- Adobe tutorial *AEM Dispatcher Common Logs*: https://experienceleague.adobe.com/en/docs/experience-manager-learn/ams/dispatcher/common-logs
