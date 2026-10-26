# ACS AEM Tools Explain Query — deprecated, with one surviving reason to use it

[Feature page](https://adobe-consulting-services.github.io/acs-aem-tools/features/explain-query/index.html). Part of **ACS AEM Tools** (the dev/one-off-tooling sibling of ACS AEM Commons — same split as acs-aem-samples: not for production installs). UI at `/etc/acs-tools/explain-query.html` (Tools → ACS AEM Tools → Explain Query); requires AEM 6.x on Oak (TarMK/MongoMK).

What it does: runs a query through the **Oak query engine's own explanation** — which index the query binds to, the actual plan — with optional real execution for timing, **slow-query warnings**, a **click-to-load list of the instance's Slow and Popular queries** (pulled from the same Oak QueryStats data the Operations dashboard reads), Oak query-plan log messages inline (v0.0.18+), and **JSON export** of the result.

## Status: deprecated — and the project says so itself

AEM 6.1+ ships the same capability natively: **Tools → Operations → Diagnosis → Query Performance → Explain Query** on 6.5 ([operations-dashboard.md](operations-dashboard.md)), and the **Developer Console query tooling on AEMaaCS** ([aemaacs-repository-inspection-by-tier.md](../cloud-service/aemaacs-repository-inspection-by-tier.md)). Default answer for "how do I see which index a query uses" is the OOTB tool, not installing ACS Tools.

## The one differentiator: QueryBuilder map explanation

Since v0.0.22 the ACS tool accepts **QueryBuilder predicate maps** (`type=cq:Page`, `path=/content`, …) as an input language — the OOTB Explain Query takes only XPath / JCR-SQL / JCR-SQL2. Without it, explaining a QueryBuilder query means first extracting the generated XPath (QueryBuilder debugger at `/libs/cq/search/content/querydebug.html`, or the `com.day.cq.search` DEBUG log) and pasting that into the OOTB tool — the ACS tool collapses those two steps. If a team lives in QueryBuilder (most AEM 6.x project code does), that's the remaining reason this deprecated tool still gets installed on dev instances.

Related: index-side view of the same problem in [oak-indexing-aemaacs.md](../cloud-service/oak-indexing-aemaacs.md) (query-explain as index validation step); the Slow Queries source MBean and traversal-limit health checks in [operations-dashboard.md](operations-dashboard.md).

Source: ACS AEM Tools feature page. Docs-based.
