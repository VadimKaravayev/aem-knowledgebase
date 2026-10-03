# Cloud Manager API: shape (HAL + events), aio CLI, Node.js/Java SDKs, GitHub Action

Everything in the Cloud Manager UI is also reachable through the **Cloud
Manager API** (`cloudmanager.adobe.io`), and Adobe ships four open-source
layers on top of it. Pick by where the call originates. All of them need an
**API integration project in the Adobe Developer Console** first, with an
**OAuth Server-to-Server** credential (client ID + secret; the older JWT
credential with a private key is end-of-life, see
[[adobe-ims-authentication-types]]). The credential's **product profile is
its Cloud Manager role**; that, not the human who created it, decides what
the calls may do.

Companion notes: [[cloud-manager-pipeline-steps]], [[app-builder]] (the usual
home for the Node SDK), [[cloud-manager-custom-domain-names]] (the domain
API calls).

---

## 1. What the API is: two halves

| Half | Direction | Purpose | Getting started |
|---|---|---|---|
| **HTTP API** | inbound, you call Adobe | read and manipulate programs, environments, pipelines, executions, domains, certificates | Developer Console **API integration** |
| **Event system** | outbound, Adobe calls you | notified when pipeline executions start, finish, wait for approval, etc. | Developer Console **Event integration** (I/O Events webhook or journaling) |

Most real integrations use both (call to start an execution, listen to learn
it finished), but Adobe's advice is to get one working first. Polling the
HTTP API for status is the fallback when a webhook endpoint is not an option.

### Creating the API integration (Developer Console)

Who may do it: a **System Administrator** or an assigned **API Developer**
of the IMS org. That *API Developer* role is a Developer Console permission
and is **unrelated to the "Developer - Cloud Manager" product profile**;
having one grants nothing about the other. Classic exam distractor.

Steps, `https://developer.adobe.com/console`:

1. **Create new project** (or open one); optionally title and describe it.
2. **Add to Project → API → Experience Cloud → Cloud Manager → Next.**
3. Choose **OAuth Server-to-Server** (the "generate or upload a key pair"
   JWT option is deprecated).
4. Pick **one Product Profile**: this maps the integration to a **Cloud
   Manager role** (Business Owner, Deployment Manager, Developer, Program
   Manager…). An API Developer may see a restricted list of profiles.
5. **Save configured API.**

What you take away: **Client ID** (a.k.a. API key) and **Organization ID**
go on every call (`x-api-key`, `x-gw-ims-org-id`); **Client Secret** and
**Scopes** are only used to mint the bearer token. Adobe's own page gives an
earlier JWT cut-off ("stop working after Jan 1, 2025") than the IMS guide's
30 June 2025 end-of-life; either way it is gone.

### Which product profile for which API operation

From Adobe's API permissions guide. The service account is org-owned, and
**cannot log into Cloud Manager or any Experience Cloud UI** — profile
assignment is purely an API-rights decision. Baseline rule: **read-only
(`GET`) access needs only the Developer profile**, with a few exceptions;
mutations follow this matrix:

| Operation | Profile(s) |
|---|---|
| Create / delete program | **Business Owner** (delete: BO only) |
| Start / advance pipeline execution | Business Owner, Deployment Manager, Program Manager |
| Certificates (create/update/delete) | Deployment Manager, Business Owner |
| Domain names | Deployment Manager, Business Owner |
| Content Copy | Deployment Manager |
| Environment logs | Deployment Manager, Developer |
| RDE reset | **Developer** |
| Environment/pipeline variables | Deployment Manager (a `403` from `aio … set-variables` = the caller lacks this role) |

Custom profiles with custom permission sets are also supported as an
alternative to the four stock roles. Per-command check from the CLI:
`--permissions` ([[aio-cloudmanager-cli]]).

### HTTP API resource structure: HAL

Resources follow the **Hypertext Application Language** convention. Each has
three parts: plain **properties** (`name`, `status`, `trigger`), **`_links`**
to related resources keyed by a *rel*, and **`_embedded`** child resources
inside list responses.

```json
"_links": {
  "self":                                        { "href": "/api/program/4/pipeline/1",            "templated": false },
  "http://ns.adobe.com/adobecloud/rel/program":  { "href": "/api/program/4",                       "templated": false },
  "http://ns.adobe.com/adobecloud/rel/executions": { "href": "/api/program/4/pipeline/1/executions", "templated": false }
}
```

- Rels are **full URIs** under `http://ns.adobe.com/adobecloud/rel/…`, plus
  the special `self`. Some links are **URI templates** (`templated: true`);
  some are arrays.
- **Embedded** resources (for example each pipeline inside
  `_embedded.pipelines`) may be a **reduced representation**; follow `self`
  for the full one. Pipeline properties visible in the embed: `id`,
  `programId`, `name`, `trigger` (`MANUAL`/…), `status` (`IDLE`/…),
  `updatedAt`, `lastStartedAt`, `lastFinishedAt`.
- Design consequence: **navigate by rel, do not hand-build URLs**. The
  domain-names flow in [[cloud-manager-custom-domain-names]] is the same
  pattern (`…/rel/domainNames`, then `…/domain-name/verify` and `/deploy`).

### Event structure: Activity Streams

Events are JSON per the **Activity Streams** spec wrapped in an envelope:

```json
{ "event_id": "3dd172b8-…",
  "event": {
    "@id":   "urn:oeid:cloudmanager:bc901126-…",
    "@type": "https://ns.adobe.com/experience/cloudmanager/event/started",
    "activitystreams:published": "2021-08-23T08:37:41.846Z",
    "activitystreams:to":     { "@type": "xdmImsOrg", "xdmImsOrg:id": "…@AdobeOrg" },
    "activitystreams:object": { "@id":   "https://cloudmanager.adobe.io/api/program/1/pipeline/2/execution/3",
                                "@type": "https://ns.adobe.com/experience/cloudmanager/pipeline-execution" },
    "xdmEventEnvelope:objectType": "https://ns.adobe.com/experience/cloudmanager/pipeline-execution" } }
```

- `@type` says **what happened** (`…/event/started`, `…/event/ended`,
  `…/event/waiting`), `activitystreams:object.@type` says **to what**
  (`pipeline-execution`, `execution-step-state`), and `object.@id` is the
  HAL URL to call back for details. The event carries **no payload beyond
  identity**; the consumer fetches the resource.
- `activitystreams:to` names the IMS org, which is how a multi-tenant
  listener routes events.

## 2. Tooling on top of the API

| Layer | Form | Typical caller | Example |
|---|---|---|---|
| **Cloud Manager CLI** | plugin for the Adobe I/O CLI (`aio`) | humans, shell scripts, CI jobs | `aio cloudmanager:list-programs` |
| **Node.js SDK** | npm library; the CLI is built on it | App Builder actions, Electron/Node tools | `const client = await sdk.init(orgId, apiKey, token); await client.listPrograms()` |
| **Java SDK** | JVM library over the same API | Java integrations, Jenkins plugins | `new CloudManagerApiImpl(orgId, apiKey, token).listPrograms()` |
| **GitHub Action** | wraps the Node SDK to **start a pipeline execution** | GitHub-hosted repos that want CM in their workflow | `adobe/aio-cloudmanager-create-execution-action` |

GitHub Action inputs, all from repository secrets: client ID, client secret,
technical account ID, IMS org ID, private key, program ID, pipeline ID (the
private-key input dates from the JWT era; check the action's current release
for OAuth S2S support before adopting it). It triggers the execution; it does not replace the Cloud Manager pipeline, which
still builds from the repository Cloud Manager knows about.

## 3. What people use it for

- Triggering or polling **pipeline executions** from an external CI/CD
  orchestrator (Jenkins, GitHub Actions, GitLab) so Cloud Manager stays the
  deployer but not the scheduler.
- Bulk or scripted administration: listing programs/environments, downloading
  logs, managing variables, IP allow lists, **custom domains and
  certificates** (the `domainNames` endpoints with `dnsTxtRecord`).
- App Builder apps that surface Cloud Manager state to a wider audience
  without granting them Cloud Manager access.

## 4. Design-review notes

- The API does not bypass Cloud Manager's quality gates; a triggered
  execution runs the same steps and approvals.
- Credentials are per integration project; scope them to a technical account
  with the least Cloud Manager role that works, and rotate the client secret
  like any other secret.
- Event consumers must be **idempotent** and tolerate reordering; use
  `event_id` for de-duplication and always re-fetch the object rather than
  trusting cached state.
- The tools are also Adobe's reference implementations of the API; when the
  REST documentation is ambiguous, read the CLI source.

## Exam checklist

- Cloud Manager API = **inbound HAL HTTP API + outbound I/O Events**; usual
  integration uses both, start with one.
- HAL: properties + `_links` (rel URIs, `self`, templated) + `_embedded`
  (possibly reduced; follow `self`). Navigate by rel.
- Events: Activity Streams JSON; `@type` = what happened, `object.@id` = the
  HAL resource to fetch; no payload beyond identity.
- Developer Console **API Developer** role ≠ Cloud Manager **Developer**
  product profile; the integration's product profile = its Cloud Manager role.
- Client ID + org ID on every call; secret + scopes only for the token.
- Cloud Manager CLI = **aio plugin**; needs a Developer Console API project.
- Node SDK powers the CLI and is the fit for **App Builder**; Java SDK for JVM.
- GitHub Action = **create a pipeline execution** from a GitHub workflow.
- "Integrate Cloud Manager with existing enterprise CI/CD" → API/CLI to start
  and watch executions, not a custom deployer.

## References
- [Understanding the API (developer.adobe.com)](https://developer.adobe.com/experience-cloud/cloud-manager/guides/getting-started/understanding-the-api)
- [Creating an API integration project (developer.adobe.com)](https://developer.adobe.com/experience-cloud/cloud-manager/guides/getting-started/create-api-integration)
- [API permissions (developer.adobe.com)](https://developer.adobe.com/experience-cloud/cloud-manager/guides/getting-started/permissions)
- [CLI and SDKs (developer.adobe.com)](https://developer.adobe.com/experience-cloud/cloud-manager/cli-and-sdks/)
- [Cloud Manager API reference (developer.adobe.com)](https://developer.adobe.com/experience-cloud/cloud-manager/reference/api/)
- [aio-cli-plugin-cloudmanager (GitHub)](https://github.com/adobe/aio-cli-plugin-cloudmanager)
- [aio-cloudmanager-create-execution-action (GitHub)](https://github.com/adobe/aio-cloudmanager-create-execution-action)
