# ACS AEM Samples: what is in it, and which samples not to copy onto AEMaaCS

**ACS AEM Samples** is a community-maintained repository of heavily commented
skeleton implementations of AEM building blocks: services, servlets, filters,
jobs, models, Query Builder extensions, replication hooks, resource
providers, workflow processes. Adobe's connector guide points ISVs at it as
reference code ([[aem-connector-implementation-guidelines]]). Facts that
matter before you trust a sample:

- It is **not a product**. Nothing in it provides real functionality and the
  package is **not vetted for unobtrusiveness**; install it **only on
  development** instances, or better, just read the code on GitHub.
- **Not supported by Adobe Support.** The GitHub org name is legacy; Adobe
  Consulting Services no longer maintains it, the community does. Issues go
  to the GitHub project.
- Siblings: **ACS AEM Commons** (real, installable toolkit: HTTP Cache,
  Generic Lists, Redirect Manager, MCP…) and **ACS AEM Tools** (developer
  utilities). Do not confuse "sample" with "commons".
- All Java lives under
  `core/src/main/java/com/adobe/acs/samples/<area>/`, mostly in `impl`
  subpackages, one class per pattern.

Companion notes: [[server-sent-events-in-aem]], [[sling-path-matches-relative-throws]],
[[httpclient-retries-timeouts-aem-connector]], [[oak-indexing-aemaacs]]
(before writing Query Builder extensions, check the index).

---

## Lookup table: "I need to…" → sample

| Need | Sample (class or package under `com.adobe.acs.samples`) | AEMaaCS note |
|---|---|---|
| Get a **service ResourceResolver** the right way | `authentication.impl.SampleServiceLoginResourceResolverImpl` | current pattern: service user + `ResourceResolverFactory.getServiceResourceResolver`, service-user mapping via repoinit |
| Hook into **login** | `authentication.impl.SampleLoginHookAuthenticationHandler` | author on AEMaaCS is IMS-only; custom auth handlers are a **publish-side** topic |
| OAuth **scope with privileges** | `authentication.oauth.impl.SampleScopeWithPrivileges` | classic Granite OAuth server; rarely used on Cloud |
| React to **Sling events** (resource/replication) | `events.impl.SampleSlingEventHandler` | handlers run on **every pod**; keep them tiny and hand off to a job |
| React to **JCR observation** | `events.impl.SampleJcrEventListener` | **avoid on AEMaaCS**: per-node, not cluster-aware, no delivery guarantee; use Sling jobs / workflow launchers |
| Event → **guaranteed work** | `jobs.impl.SampleEventHandlerWithImmediateJobExecution`, `jobs.impl.SampleSlingJobConsumer` | the **right** async pattern: `JobConsumer` with a topic, idempotent, retried |
| **Scheduled** code | `schedulers.impl` | on AEMaaCS prefer **Sling Scheduled Jobs** over `Scheduler`/`@Component` whiteboard schedulers: jobs survive pod loss ([[aem-connector-implementation-guidelines]]) |
| **Servlet filter** at Felix / Sling request / Sling include level | `filters.impl.SampleServletFilter`, `SampleSlingRequestFilter`, `SampleSlingIncludeFilter`, `SampleThreadLocalFilter` | mind the scope: an include filter runs per component include; the thread-local sample shows request-scoped state without static leaks |
| **Servlet** by resource type | `servlets.impl.SampleAllMethodsServlet`, `SampleSafeMethodsServlet` | register by `resourceTypes`, never by path, or the dispatcher and ACLs cannot protect it |
| **Sling Model** / **Model Exporter** / **Content Services**-compatible model | `models.SampleSlingModel`, `SampleSlingModelExporter`, `SampleComponentExporter` | the exporter pair is the headless/SPA Editor JSON contract |
| Custom **`adaptTo`** | `adapterfactories.impl` | declare `adaptables`/`adapters` properties so the Adapter Manager finds it |
| Fake or extend resources: **Resource, Wrapper, Decorator, Visitor** | `resources.SampleResource`, `SampleResourceWrapper`, `impl.SampleResourceDecorator`, `SampleResourceVisitor` | decorators run on every resolution; keep them O(1) |
| Expose **non-JCR data as resources** | `resourceproviders.impl` | `ResourceProvider` (Sling 2.x API); combine with a Sling Model for rendering external systems without importing them |
| **Query Builder**: nearly every predicate, custom hit writer, filtering evaluator, facet extractor | `search.querybuilder.impl.SampleQueryBuilder`, `SampleJsonHitWriter`, `SampleFilteringPredicateEvaluator`, `SampleFacetPredicateEvaluator` | a filtering evaluator post-filters **in memory**; unbounded result sets will hurt, add an index-backed predicate first |
| **Replication** hooks | `replication.impl.SampleReplicationContentFilter`, `SampleReplicationPreprocessor` | classic replication agents; on AEMaaCS publishing is Sling Content Distribution, these hooks only matter for 6.5/AMS or preview/dispatcher-flush agents |
| **Workflow** process step (Granite, CQ, wrapping) | `workflow` package: Granite `WorkflowProcess`, CQ `WorkflowProcess`, wrapper | write **Granite** (`com.adobe.granite.workflow`) processes; the CQ API sample is legacy |
| **Task Management** (Inbox tasks) | task management sample | `TaskManager` API for human-in-the-loop steps |
| **JMX MBean** with stats and operations | `mbean.impl.SampleMBeanImpl` | visible in the Felix console; on AEMaaCS there is no JMX console, expose via logs/health checks instead |
| **Health Check** / Operations Dashboard overlay | health check sample | Sling Health Checks feed Cloud Manager and the Developer Console |
| **OSGi service** patterns: R6 annotations, multi-reference, mutable state, basic | `services.impl.SampleOsgiR6AnnotationsImpl`, `SampleMultiReferenceServiceImpl`, `SampleMutableStateServiceImpl`, `SampleServiceImpl` | R6/R7 DS annotations are the only style to copy; the mutable-state sample shows why services must be thread-safe |
| **Predicate** for path fields / lists | `predicates.impl.SamplePredicate` | classic `com.day.cq.commons.predicate`, still used by Granite pickers |
| Touch UI **Data Source** for dialogs | UI widgets sample | the standard way to feed dynamic options into a `select` |
| HTL page implementation, JS Use object, custom binding provider | HTL samples (heervisscher) | `BindingsValuesProvider` is the supported way to inject globals into HTL |
| **Header/footer** as a separate, cacheable, personalised fragment | Header/Footer WCM solution | Sling Dynamic Include + ACS Commons HTTP Cache; the concept survives on AEMaaCS as Experience Fragments + CDN/dispatcher caching |

## Reading order for someone new to AEM back-end code

1. `services.impl.SampleOsgiR6AnnotationsImpl` (how anything is wired).
2. `authentication.impl.SampleServiceLoginResourceResolverImpl` (how to get
   repository access without `admin`).
3. `models.SampleSlingModel` → `SampleSlingModelExporter` (rendering and JSON).
4. `servlets.impl.SampleSafeMethodsServlet` (HTTP entry point done right).
5. `jobs.impl.SampleSlingJobConsumer` (async done right).
6. `search.querybuilder.impl.SampleQueryBuilder` (finding content).

## Exam and review checklist

- ACS **Samples** = teaching skeletons, dev-only, community-maintained, no
  Adobe support. ACS **Commons** = the real toolkit.
- Async work in AEM: **Sling Jobs** (guaranteed, retried) over event
  handlers or JCR listeners (fire-and-forget, per-instance).
- Servlets bind to **resource types**, not paths.
- Repository access from services: **service users**, never `admin` sessions
  or `loginAdministrative`.
- Replication hooks and JMX MBeans are 6.5/AMS-era concerns; on AEMaaCS the
  equivalents are content distribution and health checks/logs.

## References
- [ACS AEM Samples site](https://adobe-consulting-services.github.io/acs-aem-samples/)
- [ACS AEM Samples on GitHub](https://github.com/Adobe-Consulting-Services/acs-aem-samples)
- [ACS AEM Commons](https://adobe-consulting-services.github.io/acs-aem-commons/)
- [ACS AEM Commons HTTP Cache](https://adobe-consulting-services.github.io/acs-aem-commons/features/http-cache/index.html)
