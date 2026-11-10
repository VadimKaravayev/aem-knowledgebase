# API gateway, BFF and rate limiting

The components that sit between clients and backend services: one entry point for everyone, a tailored backend per kind of client, and the limits that protect what is behind them. Docs-based; the token bucket sample was compiled and run on Java 23. Related: [[api-design]] (REST, idempotency, versioning), [[security-auth]] (tokens), [[messaging-patterns]] (asynchronous integration), [[reliability-patterns]].

## API gateway

A single entry point in front of many services. Clients talk to the gateway; the gateway forwards to whichever service owns the request.

```
web  ─┐                      ┌──▶ order-service
app  ─┼──▶  API gateway  ────┼──▶ customer-service
partner┘                     └──▶ catalogue-service
```

What it does, so that each service does not have to:

| Concern | At the gateway |
|---|---|
| Routing | Path, host or header → service; versions; canary and blue-green splits |
| TLS | Terminates client connections |
| Authentication | Validates the token or API key once, at the edge |
| Rate limiting and quotas | Per client, per plan |
| Transformation | Header rewriting, protocol translation (REST ↔ gRPC), response shaping |
| Aggregation | Combining several backend calls into one response |
| Cross-cutting | CORS, caching, request size limits, access logs, metrics, trace start |

Why have one: clients need one address and one security model instead of knowing every service; services can be split, merged and moved without clients noticing; and policy is applied in one place.

Traps:

- **It is on the path of every request.** It must be redundant and scaled like the most critical service, and it adds a hop of latency to all of them.
- **Business logic creeps in.** A gateway that validates orders, joins data and holds rules has become a monolith in front of the microservices. Keep it to routing and cross-cutting policy; put composition in a BFF or a service.
- **Gateway authentication is not sufficient.** Services should still verify the token or a trusted identity passed by the gateway — anything that reaches a service by another path is otherwise unauthenticated.
- **One gateway, one team, every change through them**: the organizational bottleneck. Let teams own their routes through reviewed configuration.
- **Timeouts must be ordered**: the gateway's timeout longer than the backend's, shorter than the client's. Otherwise the gateway reports a timeout for work that later succeeds.

Related components, often confused with it:

| | Faces | Purpose |
|---|---|---|
| Load balancer | One service | Spread traffic over that service's instances |
| Reverse proxy | Anything | Forwarding, TLS, caching — the mechanism a gateway is built on |
| **API gateway** | External clients ("north–south") | API-level policy: identity, limits, routing by API |
| Service mesh | Service-to-service ("east–west") | Encryption, retries and telemetry between services ([[distributed-tracing]]) |

Products: Kong, Apigee, AWS API Gateway, Azure API Management, Spring Cloud Gateway, and gateways built on NGINX or Envoy.

## Backend for frontend (BFF)

A **separate backend for each kind of client** — one for the web application, one for the mobile app, one for partners — each exposing exactly the API that client needs.

```
web app    ──▶  web BFF     ─┐
mobile app ──▶  mobile BFF  ─┼──▶  order, customer, catalogue services
partners   ──▶  partner API ─┘
```

The problem it answers: one general-purpose API serves every client badly. A mobile screen needs three fields and gets forty; it needs data from five services and makes five round trips on a slow network; and any change for one client risks breaking the others.

What a BFF does:

- **Aggregates** the calls a screen needs into one request, made in parallel server-side.
- **Shapes** the response for that client: fewer fields and smaller images for mobile, a different structure for the web.
- **Handles that client's authentication flow.** For browser applications, the BFF keeps the access and refresh tokens on the server and gives the browser only a session cookie that scripts cannot read — the recommended way to keep tokens out of the browser ([[security-auth]]).
- **Absorbs backend change**, so a service can evolve without every released mobile version breaking.

It is normally **owned by the team that builds that frontend**, which is the point: they change client and BFF together, without waiting for a platform team.

Traps:

- **Duplication between BFFs.** The same logic appearing in two of them belongs in a downstream service, not in a shared BFF library that couples them again.
- **Business logic in the BFF.** It should hold presentation logic only: what to fetch and how to arrange it. Rules about orders live in the order service.
- **Too many of them.** One per client *experience*, not one per screen or per platform — iOS and Android usually share one.
- **Fan-out latency and failure.** Five parallel calls are as slow as the slowest and fail if any fails. Each needs a timeout, and the response should degrade — return what arrived — rather than fail whole ([[reliability-patterns]]).
- It is one more deployable to run, secure and monitor per client type.

Gateway and BFF combine: the gateway in front does the generic work (TLS, limits, routing), the BFFs behind it do the client-specific work. **GraphQL** is the main alternative — one endpoint where each client states the shape it wants — which moves the tailoring from server code to client queries.

## Rate limiting

Capping how many requests a client may make in a period, and rejecting the excess.

Why: to protect services from overload and from one client starving the others; to contain abuse (credential stuffing, scraping); to control the cost of expensive operations; and to enforce commercial plans.

### Algorithms

| Algorithm | How it works | Behaviour |
|---|---|---|
| **Fixed window** | Count requests per clock window (per minute); reset at the boundary | Simple. Allows **twice the limit** in a short span straddling a boundary |
| **Sliding window log** | Keep the timestamp of each request; count those in the last N seconds | Exact. Memory grows with the request rate |
| **Sliding window counter** | Weighted blend of the current and previous window's counts | Close to exact at fixed-window cost. The usual compromise |
| **Token bucket** | Tokens are added at a steady rate up to a capacity; each request takes one | Allows **bursts** up to the capacity, then the steady rate. The common default |
| **Leaky bucket** | Requests queue and are released at a constant rate | Smooths output completely; adds delay; excess overflows |

```java
public final class TokenBucket {
    private final long capacity;
    private final double refillPerNano;
    private double tokens;
    private long last;

    public TokenBucket(long capacity, double refillPerSecond) {
        this.capacity = capacity;
        this.refillPerNano = refillPerSecond / 1_000_000_000.0;
        this.tokens = capacity;
        this.last = System.nanoTime();
    }

    public synchronized boolean tryAcquire() {
        long now = System.nanoTime();
        tokens = Math.min(capacity, tokens + (now - last) * refillPerNano);
        last = now;
        if (tokens < 1) return false;
        tokens -= 1;
        return true;
    }
}
```

With a capacity of 10 and a refill of 5 per second, a burst of 100 calls gets 10 through; a second later 5 more are allowed. In practice use a library (Bucket4j, Resilience4j) or the gateway's built-in limiter.

### Design decisions

- **What to key on**: API key, user, tenant, IP address or a combination. **Per-IP punishes everyone behind a shared address** (an office, a mobile carrier) and is trivially evaded by attackers with many addresses; use it only for unauthenticated traffic. Per-tenant is what protects a multi-tenant platform.
- **Where**: at the edge (gateway or CDN) so rejected requests cost almost nothing; additionally inside a service for limits only it understands (per expensive operation).
- **How to answer**: `429 Too Many Requests` with a `Retry-After` header, and headers that show the client its limit and remaining allowance so it can pace itself.
- **Across many gateway instances** the counter must be shared — typically in Redis, with the check-and-decrement done atomically. Counters kept per instance multiply the real limit by the number of instances. A shared limiter that is slightly approximate is better than an exact one that is slower than the service it protects.
- **When the limiter's store is down**: fail open (allow, and lose protection) or fail closed (reject everyone). Decide in advance; for most APIs, fail open with a local fallback limit.

### Related controls

| | Limits | Purpose |
|---|---|---|
| Rate limit | Requests per second or minute | Protection against bursts |
| Quota | Requests per day or month | Commercial plans, fair use |
| Concurrency limit | Requests in progress at once | Protection that adapts when the backend slows down |
| Throttling | Slows or queues instead of rejecting | Smoothing |
| Load shedding | Rejects by priority when the server is saturated | Survival, regardless of who is asking |

A rate limit bounds what one client may ask for; it does not guarantee the service can serve the total. Concurrency limits and load shedding ([[reliability-patterns]]) cover that.

Traps:

- **Clients that retry immediately on `429`** turn a limit into a synchronized storm. Clients must back off with jitter and respect `Retry-After`.
- **Limits that are not documented or visible** produce support tickets instead of well-behaved clients.
- **One global limit for all endpoints**: a cheap read and an expensive export should not share an allowance.
- **Limiting after the expensive work** — authenticate cheaply, limit, then do the work.

## What this means for AEM

- **AEM is a backend behind these components, not one of them.** For headless delivery, AEM's GraphQL persisted queries play the BFF's shaping role for content: the query is defined on the server and the client fetches a cacheable URL. Combining AEM content with commerce or other APIs is done in a separate layer — Adobe's API Mesh on App Builder is the provided one ([app-builder](../cloud-service/app-builder.md)).
- **On AEMaaCS, rate limiting is configured at the Adobe-managed CDN** through traffic filter rules, so excess requests are rejected before reaching publish. The limit is counted per CDN location, which is the per-instance multiplication described above.
- On AEM 6.5 the equivalent sits in whatever is in front of the dispatcher — CDN, load balancer or web application firewall; the dispatcher itself is not a rate limiter.
