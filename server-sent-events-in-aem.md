# Server-Sent Events from an AEM servlet

Pushing live updates to an author-tier console (job progress, status, notifications) without polling.
Verified on an AEM as a Cloud Service SDK Quickstart, Sept 2026, with a Sling servlet and a Coral
progress bar.

Spec: [WHATWG HTML, Server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html).
The parsing rules are the part worth reading:
[parsing an event stream](https://html.spec.whatwg.org/multipage/server-sent-events.html#parsing-an-event-stream).

## It works, and AEM does not buffer

A `SlingSafeMethodsServlet` that writes frames and flushes each one streams them immediately. Ten
frames written a second apart arrived a second apart. Response headers:

```
Content-Type: text/event-stream;charset=utf-8
Cache-Control: no-cache
Transfer-Encoding: chunked
```

Per frame, both calls are needed:

```java
writer.flush();
resp.flushBuffer();
```

`text/event-stream` is not optional. It is registered with IANA and `EventSource` enforces it: any
other content type and the browser parses nothing and fires `error`.

**`X-Accel-Buffering: no` is nginx-only.** It does nothing for Jetty, Apache or the AEM dispatcher.
Do not cargo-cult it in.

## Registration and auth

A resource-type-bound servlet answers on any extension you name, so `extensions = "sse"` gives
`/content/services/<app>/<node>.sse` with no resolver configuration. The mount node is an ordinary
`nt:unstructured` node carrying `sling:resourceType`.

**`EventSource` cannot send request headers**, so no `Authorization`. On the author tier this is a
non-issue: the request carries the login-token cookie like any other same-origin GET. Anything
needing a bearer token has to use `fetch` with a streaming body reader instead.

## The cost: one request thread per stream

Measured with 8 concurrent streams on an idle author:

| | Total threads | Threads on the endpoint |
|---|---|---|
| Idle | 262 | 0 |
| 8 streams | 264 | 8 |
| Closed | 262 | 0 |

Exactly 1:1. The thread is parked in a blocking `poll`, not spinning, so the CPU cost is nil and the
pool slot is the whole cost. Jetty reused idle pool threads for 6 of the 8.

Two consequences for a real deployment:

- The pool is shared with every other request. Enough open consoles and ordinary requests queue
  behind streams. Cap concurrent streams, or use `request.startAsync()` to hand the thread back.
- A long-lived request holds its `ResourceResolver` and the JCR session behind it. An Oak session
  pins the revision it was opened at, which interferes with revision GC. Do not touch the repository
  on the stream thread, and do not let a stream live for hours.

**Reading a thread dump:** Sling renames the request thread to the request line while it processes,
so busy threads vanish from a `qtp` grep. Count them by endpoint:

```bash
curl -s -u admin:admin http://localhost:4502/system/console/status-Threads.txt | grep -c '<your-endpoint>'
```

## The trap: a stream with no lifecycle exit is immortal

A loop whose only exits are "client disconnected", "work finished" and "interrupted" **survives a
bundle stop**. Observed: bundle `Resolved`, request thread still alive, browser still connected,
progress frozen forever. Restarting the bundle made it worse. The thread holds a reference to the
old service instance from the stopped bundle, while the new producer publishes into the new
instance. The stream is orphaned permanently and only the client can end it.

Worse, the browser cannot detect this. Keepalive comments (`:\n\n`) keep arriving from the orphaned
loop, so the connection looks healthy. A frozen bar is indistinguishable from a slow one.

The fix is a flag the loop checks, plus a hard cap:

```java
private final AtomicBoolean running = new AtomicBoolean(true);

@Deactivate
protected void deactivate() {
    running.set(false);
}

while (running.get() && (System.currentTimeMillis() < deadline)) { ... }
```

Detection latency equals the poll timeout, because the thread only re-checks the flag when `poll`
returns. With a 15-second poll, a bundle stop is noticed within 15 seconds. **Do not shorten this
with an interrupt:** Cloud Manager bans `Thread.interrupt()` in every form (`CQRules:GRANITE-54181`),
so a checked flag is the only compliant mechanism.

## Reconnection is weaker than it looks

`EventSource` retries automatically after a dropped connection, roughly 3 seconds by default, and
sends `Last-Event-ID` so the server can resume. That covers a network blip and nothing else.

**A non-200 response fails the connection permanently.** `readyState` becomes `CLOSED` and the
browser never tries again. During a redeploy the servlet is unregistered, the resource has no
handler, and Sling answers 404, so the single automatic retry lands on a 404 and the page is dead
until someone reloads it. Observed exactly that: bar stuck at 26 through the restart and beyond.

Any console that must survive a deploy needs its own retry:

```js
source.addEventListener("error", function () {
    if (source.readyState !== EventSource.CLOSED) {
        return;   // the browser's own retry is still running; racing it opens two streams
    }
    window.setTimeout(function () { connect(url, Math.min(delay * 2, MAX_MS)); }, delay);
});
```

With that in place a 45-second outage produced four 404 retries and then a clean 200, and the page
recovered by itself. Note it recovers the *stream*, not the *run*: a new subscription starts from
zero unless the server honours `Last-Event-ID`.

## Shape that survives contact with a real dashboard

A dashboard carries more than one kind of event, and the six-connections-per-origin limit on
HTTP/1.1 makes a stream per widget expensive. Use one channel per dashboard with named events, and
an envelope so the transport never changes:

```java
@Value @Builder
public class DashboardEvent {
    @NonNull String name;      // the SSE "event:" line
    @NonNull Object data;      // serialized to "data:" by Jackson
    boolean terminal;          // the only thing the servlet needs to know
}
```

`terminal` is what lets the servlet close a stream without knowing what any event means. Adding an
event type then touches one payload class, one producer and one client listener; the publisher and
the servlet are untouched.

Producer and consumer hand off through a bounded queue per subscriber, `ConcurrentHashMap<String,
BlockingQueue<DashboardEvent>>`:

- **`offer`, never `put`.** The producer is usually a single scheduled thread serving every
  subscriber. `put` blocks when a queue is full, so one stalled browser would stall everyone.
- The queue is also what keeps the producer off the response writer, which belongs to the request
  thread.
- `synchronized` is banned by Cloud Manager (AEM-15); the concurrent collections cover this
  completely.

For the periodic producer, a `@Component(service = Runnable.class)` with `scheduler.period` is the
AEM-native mechanism, and `scheduler_period()` in an `@ObjectClassDefinition` makes it tunable live
from `/system/console/configMgr`. No `ScheduledExecutorService`, which would drag in the banned
`shutdownNow()`.

## Multi-pod author: an in-memory publisher reaches one pod

- **Symptom:** on AEMaaCS, some authors' dashboards never update while others do; fine on a local Quickstart. (Derived from the platform, not yet reproduced.)
- **Root cause:** author runs at least two pods. A publisher held in memory only reaches streams parked on the pod where the job runs.
- **Fix:** the job writes state to the JCR; on every pod a `ResourceChangeListener` + `ExternalResourceChangeListener` on that subtree re-reads the path and publishes locally. Send a JCR snapshot on connect.
- **How to spot it:** any SSE design where the producer calls the publisher directly. Ask which pod holds the stream.

## Checklist

1. `text/event-stream`, UTF-8, `Cache-Control: no-cache`.
2. `writer.flush()` **and** `resp.flushBuffer()` per frame.
3. `@Deactivate` flag checked by the loop, plus a maximum duration.
4. Keepalive comment on poll timeout.
5. `unsubscribe` in a `finally`.
6. Client-side retry with backoff for the 404-during-deploy case.
7. Cap concurrent streams, or move to `startAsync()`.
8. Never touch the repository from the stream thread.
9. Publish from a JCR listener with the external marker, never directly from the producer.

Related to 9: [translation-connector-cached-services.md](translation-connector-cached-services.md).

Related: [multi-author-scaling.md](multi-author-scaling.md),
[osgi-import-package-version-range.md](osgi-import-package-version-range.md),
[granite-i18n-client-snippets.md](granite-i18n-client-snippets.md).
