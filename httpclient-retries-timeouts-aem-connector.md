# Apache HttpClient 4.5 in an AEM connector: retries, timeouts, per-call policy

Every AEM connector that talks to a vendor API needs the same four things: a pooled client, timeouts,
retries, and a way to let one call behave differently from the rest. HttpClient 4.5 provides all four,
and the reflex to write a `for` loop with `Thread.sleep` produces code that looks correct and is not.

## Use what AEM already exports

`org.apache.http.*` (HttpClient 4.5) is on **both** `aem-sdk-api` and `uber-jar` 6.5.22. Verified
Sept 2026:

```bash
unzip -l uber-jar-6.5.22.jar | grep -E 'org/apache/http/(client/ServiceUnavailableRetryStrategy|impl/client/DefaultHttpRequestRetryHandler)\.class'
```

So no vendored HTTP client, no embedded jar, no class-file-version risk. Add one line to the bnd
instructions, per [osgi-import-package-version-range.md](osgi-import-package-version-range.md):

```
Import-Package: ...,org.apache.http.*;version=0.0.0,...
```

## The two retry hooks

| Failure | Hook | Set on |
|---|---|---|
| Connection failure, i.e. an `IOException` before any response | `HttpRequestRetryHandler` | `HttpClientBuilder.setRetryHandler` |
| A response you dislike, i.e. `429` and `5xx` | `ServiceUnavailableRetryStrategy` | `HttpClientBuilder.setServiceUnavailableRetryStrategy` |

Never retry `4xx`. It is the answer, not a glitch.

Extend `DefaultHttpRequestRetryHandler` rather than implementing the interface: it already knows which
exceptions are hopeless (`UnknownHostException`, `SSLException`, …) and which methods may be replayed.
Construct it with a high count and gate on your own count first, so its remaining rules still apply.

**POST semantics worth knowing before you "fix" them.** With the default
`requestSentRetryEnabled=false`, a non-idempotent request is retried only when it never reached the
server. That is why a token exchange is not silently submitted twice. Do not flip the flag casually.

## Per-call behavior on a shared client: `HttpContext`

Both hooks are configured per *client*, but the interactive and background callers want opposite
things: a screen with someone waiting wants one attempt, a status poller wants retries. Do not build
two clients. Put the caller's policy in the context:

```java
HttpClientContext context = HttpClientContext.create();
context.setAttribute(ATTR_CALL_POLICY, policy);
client.execute(request, context);
```

Both `retryRequest(...)` overloads receive that `HttpContext`, so one client serves every caller.

## The trap: `getRetryInterval()` takes no arguments

```java
public interface ServiceUnavailableRetryStrategy {
    boolean retryRequest(HttpResponse response, int executionCount, HttpContext context);
    long getRetryInterval();                    // no response, no context, no count
}
```

Consequences, and they are not obvious until you try:

- **`Retry-After` cannot be honoured.** The header is on the response, which this method never sees.
- **Exponential backoff cannot be expressed.** No execution count.
- **The interval cannot vary per call.** No context.

So with the library alone you get one fixed interval. The escape hatch is a `ThreadLocal<Long>` set
inside `retryRequest` and read in `getRetryInterval`, which is safe because HttpClient calls them
back to back on the same thread. It works; it is also state smuggled through a side channel. Decide
deliberately rather than by accident, and if a fixed interval is good enough, take it.

## Timeouts, and what they do not mean

Set them per request, not per client:

```java
request.setConfig(RequestConfig.custom()
        .setConnectTimeout(connectMs)
        .setConnectionRequestTimeout(connectMs)
        .setSocketTimeout(readMs)
        .build());
```

Two facts that break the usual mental model of "the call takes at most N seconds":

1. **Connect and read timeouts are separate and additive.** 5 s connect plus 10 s read is a 15 s call.
2. **The socket timeout is per read, not for the whole call.** A server that trickles one byte every
   9 s resets the clock each time and the call runs indefinitely.

HttpClient 4.5 has **no total-call timeout**. A real ceiling needs a watchdog that calls
`request.abort()` when the deadline passes.

**Why `abort()` and not a `Future`:** Cloud Manager's `CQRules:GRANITE-54181` bans `Thread.interrupt()`
in every form, including `Future.cancel(true)` and `ExecutorService.shutdownNow()`, because Oak
misbehaves on interrupted threads. `HttpRequestBase.abort()` closes the connection instead of
interrupting a thread, so it is allowed.

## Anti-pattern: splitting one deadline across sequential calls

A verification step that does two calls invites this:

```java
long deadline = now() + budgetMs;                       // 10 s for both
String bearer = exchange(token, once(budgetMs));        // call 1 gets all of it
int remaining = (int) (deadline - now());
if (remaining <= 0) throw timeout();                    // throws away a valid bearer
return whoAmI(bearer, once(remaining));                 // call 2 gets the leftovers
```

Three defects, all of them the arithmetic's fault:

- A 9.8 s exchange leaves `whoAmI` 200 ms, which times out, and a **healthy** connection reports
  "the service did not respond". The guard invents the failure it was meant to prevent.
- The `remaining <= 0` throw discards a successful call. The waiting already happened; throwing after
  it saves nobody any time and reports something false.
- The deadline is only checked *between* calls, which is exactly where it cannot help. Nothing
  enforces it *during* one.

**Give each call the same timeout instead.** Two calls at 5 s each is the same 10 s for the screen,
neither call can starve the other, and the code is three lines with no clock arithmetic.

## OSGi lifecycle

Build in `@Activate`, rebuild on `@Modified` so a configuration change takes effect without a restart,
close in `@Deactivate` (both the client and the connection manager). Hold the client in a `volatile`
field and read it once per call into a local, since `@Modified` can replace it mid-flight.

---

Implemented and verified on the Phrase TMS connector, Sept 2026. Related:
[java-target-platform-vs-build-jdk.md](java-target-platform-vs-build-jdk.md),
[osgi-import-package-version-range.md](osgi-import-package-version-range.md),
[translation-connector-cached-services.md](translation-connector-cached-services.md).