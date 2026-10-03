# Cloud Manager API: New Relic sub-account users

Cloud Manager programs (AMS) come with a **New Relic APM sub-account**, and
access to it is managed **through the Cloud Manager API**, not in New Relic
itself. Two operations, both program-scoped and tagged under *Programs* in
the spec; general HAL/auth conventions in [[cloud-manager-api-cli-sdks]].

---

## The endpoint pair

`/api/program/{programId}/newRelicUsers`

- **GET** — `getNewRelicSubAccountUserList`: returns
  `NewRelicSubAccountUserlist`:
  - `_totalNumberOfNewRelicSubAccountUsers` (int),
  - `applicationId` — the Cloud Manager **application** this sub-account is
    linked to,
  - `_embedded.newRelicUsers[]` — each user: `id`, `firstName`, `lastName`,
    `email`, `role` (e.g. `admin`), `owner` (boolean, default `false`),
  - HAL `_links`: `self`, `…rel/application`, `…rel/newRelicSubAccount` —
    navigate by rel as usual.
- **PATCH** (same path) — `createDeleteNewRelicSubAccountUsers`: **one
  endpoint both adds and removes users** (no POST/DELETE). Requires
  `Content-Type: application/json`. Responses: `200` with the resulting
  user list, or **`204` "Successful New Relic Sub Account user flow
  initiation"** — the mutation is an async *flow*, so a 204 means accepted,
  not completed; re-GET to confirm.

Auth = the standard Cloud Manager API trio on every call:
`Authorization: Bearer <token>`, `x-api-key` (IMS client ID),
`x-gw-ims-org-id`.

## Spec caveats

- The PATCH body is declared as a **single `NewRelicSubAccountUser`
  object**, while the operation name promises create *and* delete — the
  spec doesn't show how an entry signals removal (the published YAML is
  loose here). Treat the shape as "verify against a live call", not gospel.
- The same user schema exists twice in the spec
  (`NewRelicSub-AccountUser` / `NewRelicSubAccountUser`), identical fields
  — an artifact, not two types.

## References
- [getNewRelicSubAccountUserList (Cloud Manager API reference)](https://developer.adobe.com/experience-cloud/cloud-manager/reference/api#operation/getNewRelicSubAccountUserList)
- Raw spec: `static/api.yaml` in [AdobeDocs/cloudmanager-api-docs](https://github.com/AdobeDocs/cloudmanager-api-docs) (see LINKS.md)

Extracted from the published OpenAPI spec, Oct 2026; operations not exercised
against a live program.
