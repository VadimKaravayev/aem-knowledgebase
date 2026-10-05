# AI-specific architecture — LLM gateway, RAG pipeline, vector DB, agent/tool boundaries, models

General system-design reference. The AEM-flavored implementations live in `ai-integration/`; this doc is the architecture vocabulary.

## LLM gateway

- A single egress point between all internal apps and model providers. Earns its place through: **provider abstraction** (one API shape over OpenAI/Anthropic/Azure/Bedrock; model swap without app changes), **central key custody** (apps never hold provider keys — see [[security-auth]]), **cost attribution** (per-team/feature token metering — the FinOps unit is tokens, not requests), **rate limiting & quotas** per client (protects the org's provider-level limits from one runaway feature), **fallback/routing** (provider outage → secondary model; cheap model for easy traffic), caching (exact or semantic), and central **audit/logging** (with PII redaction before logs — prompts are user data).
- It's an API gateway with LLM-shaped traffic — the [[api-design]]/[[reliability-patterns]] rules apply with twists: responses are **streaming** (SSE; time-to-first-token and tokens/sec are the latency metrics, not request duration), timeouts must fit generation length (and distinguish "slow start" from "slow stream" — stall detection beats total-duration timeouts), retries are expensive (you pay for the failed tokens) and non-deterministic (a "retry" is a new sample, not a replay).
- Provider rate limits are in **tokens per minute**, not RPS — capacity planning and the gateway's limiter must count tokens both ways.
- Degradation ladder (circuit breaker outcomes): primary model → fallback model → cached/semantic-cache answer → graceful "AI unavailable" UX. Decide it in advance, per feature.

## RAG pipeline

- Two halves. **Ingestion (offline)**: source → extract/clean → **chunk** → embed → index (vector DB + metadata). **Query (online)**: embed query → retrieve top-k (+ filters) → optionally **rerank** → assemble context under a token budget → generate with citations.
- **Chunking is the highest-leverage, least-glamorous decision**: too small = answers split across chunks; too big = retrieval blur + wasted context. Respect semantic boundaries (headings, sections) over fixed sizes; overlap helps; store parent-document references so retrieval can expand a hit to its surroundings ("small-to-big").
- **Hybrid retrieval beats pure vector**: embeddings miss exact identifiers, SKUs, error codes — pair vector search with BM25/keyword and fuse (RRF). A cheap **reranker** (cross-encoder) over the top ~50 reliably buys more quality than a bigger embedding model.
- **Metadata filtering is load-bearing**, not an extra: tenant isolation, language, recency, and — critically — **permissions: filter by the *user's* ACLs at query time**, or RAG becomes a data-leak machine that helpfully summarizes documents the user can't open. Index-time access labels + query-time filter is the pattern.
- Freshness: embeddings are a **derived copy** of source content — updates need re-chunk/re-embed (CDC from the source of truth; the [[async-systems]] outbox problem again). An embedding-model upgrade invalidates the whole index → plan dual-index blue/green reindexing from day one.
- Evaluate the halves separately: retrieval metrics (recall@k, MRR — did the right chunk even arrive?) before generation metrics (faithfulness, answer relevance). Most "the LLM is hallucinating" bugs are retrieval misses; check what was in the context before blaming the model.

## Vector DB

- The job: approximate nearest-neighbor (ANN) search over embeddings. **HNSW** (graph-based) is the default index — fast, high recall, memory-hungry; IVF/PQ variants trade recall for RAM at large scale. ANN is *approximate*: recall is a tunable (efSearch etc.), and "vector DB returns wrong results" is often just recall tuned too low.
- Distance metric must **match the embedding model's training** (cosine vs dot vs L2 — normalized embeddings make cosine ≡ dot); mixing models in one collection is silent garbage (vectors from different models aren't comparable).
- The real differentiator between products is not ANN speed but **filtered search** done well (pre-filter vs post-filter: post-filtering top-k can return < k or 0 results after the filter — the classic "search works until you add tenant filters" bug), plus ops maturity (replication, backups).
- **You may not need a dedicated one**: pgvector (Postgres) or the search engine you already run (Elasticsearch/OpenSearch kNN) keeps vectors next to the data and ACLs, one consistency boundary fewer ([[data-storage]] polyglot rule). Dedicated vector DBs earn it at high scale or high QPS.
- It's an index, not a system of record: store canonical text/metadata elsewhere; be able to rebuild the collection from source.

## Agent / tool boundaries

- An agent = LLM in a loop choosing **tools** (functions) until done. The architecture lives at the boundary: tools are the API the model consumes — small, orthogonal, with descriptions written for the model (the description *is* the interface contract) and typed, validated inputs.
- **Treat the model as an untrusted caller**: validate every tool argument server-side, enforce authorization on the *user's* identity (the agent must not have standing permissions beyond its user — confused-deputy risk), allowlist side-effecting tools, and gate destructive/irreversible actions on human confirmation. **Prompt injection means any text the model reads (retrieved docs, web pages, emails) can steer tool calls** — the security boundary is at tool execution, not at the prompt.
- Reliability shape: the loop is non-deterministic and can run away → cap iterations/tokens/cost per task, make tools **idempotent** where possible (the model retries on its own initiative — [[api-design]] idempotency, with a stochastic client), and checkpoint long tasks.
- State: conversation/trajectory state lives server-side (the context window is the working set, not the store); summarize/compact as it grows. Multi-agent = same boundary rules per agent + explicit message contracts between them; prefer one agent + good tools until proven otherwise.
- Protocol note: **MCP** (Model Context Protocol) standardizes the tool/resource boundary between hosts and tool servers — the emerging interop layer for "agent calls org capability".

## Model layer (selection, versioning, ops)

- **Pin model versions** in production (`*-latest` aliases shift under you; provider upgrades change behavior silently) and treat a model bump like a dependency upgrade: run the **eval suite** before rollout, canary a traffic slice, keep rollback. No evals = no safe model changes; a golden dataset of real task examples + LLM-as-judge/assertions is the regression suite of this world ([[observability]] applies: log prompt/response/model-version/latency/token counts per call, with trace IDs across the pipeline).
- **Routing**: match model tier to task difficulty (cheap-fast for classification/extraction/drafts; frontier for reasoning-heavy steps). Cost control levers in order of leverage: shorter prompts/context discipline → caching (provider prompt-caching for shared prefixes; semantic caching has false-positive risk on near-duplicate queries) → model downshift → batching (provider batch APIs at ~50% price for offline work).
- Structured output: use native JSON/schema modes or tool-calling for machine-readable results; validate + repair-reprompt as the error path. Temperature 0 reduces but does **not** eliminate nondeterminism.
- Capacity: throughput is bounded by provider TPM quotas per model per region — a second region/provider is the scaling *and* the availability plan (the gateway is where that routing lives).

## AEM tie-ins

See `ai-integration/` for the applied layer. The permission-aware retrieval problem maps directly onto Oak ACLs; content freshness for RAG over AEM content is a replication-event/CDC problem ([content-distribution-journal](../cloud-service/content-distribution-journal.md)); and AEMaaCS explicitly lacks vector similarity in Oak Lucene indexes ([oak-indexing-aemaacs](../cloud-service/oak-indexing-aemaacs.md)) — embeddings for AEM content live in an external store by necessity, with App Builder as the usual middleware ([app-builder](../cloud-service/app-builder.md)).
