# UGC on AEMaaCS: App Builder is middleware, not a database host

When a component needs to persist user-generated content (ratings, comments, form submissions, etc.) on AEM as a Cloud Service, the answer is always an **external data store** — Publish is immutable/stateless and reverse replication doesn't exist there anymore (see [app-builder.md](app-builder.md)). The follow-up question is *where does that external store actually live*, and the answer is **not** "hosted by Adobe."

## Adobe does not offer a general-purpose hosted database

There is no Adobe Cloud service you provision like an AWS RDS/DynamoDB instance to hold your own structured application data. App Builder's only built-in persistence primitives are:

- **State SDK** — a key-value store with TTL-based expiry, meant for lightweight action state/caching. Small values, no relational queries/filtering — not a system of record for potentially high-volume UGC.
- **Files SDK** — blob storage for files/binaries, not structured/queryable data.

Neither is a substitute for a real database.

## The actual pattern

```
AEM component (client-side call)
        │
        ▼
App Builder I/O Runtime action  (middleware — auth, validation, business logic)
        │
        ▼
External cloud database on a mainstream provider
  (AWS RDS/DynamoDB, Azure Cosmos DB/SQL, GCP Cloud SQL/Firestore, etc.)
```

App Builder hosts the **middleware/API layer**, not the data. The action is just an ordinary client to whichever external database the customer already runs or provisions on their own cloud account.

## Exam/design trap

Don't read "use an external data store" as "Adobe hosts it for you" or "just use App Builder's State SDK and call it done." App Builder solves *where the custom code runs* (off the AEM JVM); it does not solve *where durable, queryable UGC lives* — that's still a genuine external database decision, same as any other backend system.

Confirmed via AEM Architect exam question-9 discussion, Sept 2026.
