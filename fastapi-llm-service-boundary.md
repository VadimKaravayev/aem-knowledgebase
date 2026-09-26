# Exposing an LLM contract over HTTP with FastAPI

How to put a validated LLM result behind an HTTP endpoint that a Java/AEM client can
consume as a DTO, and how to actually run the thing. Companion to
[llm-structured-output-contracts.md](llm-structured-output-contracts.md), which covers how
the model fills the schema; this file covers the boundary in front of it.

Verified on fastapi 0.141.1 / starlette 1.6.0 / uvicorn 0.53.0 / pydantic 2.13.5 /
anyio 4.15.0 / Python 3.13.2, Sep 2026.

## The three-file split

```
contracts.py   # Pydantic: the inbound request model AND the outbound result model
service.py     # the model call, the prompt, the failure translation. No HTTP, no printing.
api.py         # FastAPI app: a handler that does nothing but call the service
```

The split is the point, not ceremony. The service function stays callable from a script,
a test, or a future agent loop with no web server in sight; the HTTP layer owns no
contract of its own and no prompt.

```python
# api.py
from fastapi import FastAPI, HTTPException

from contracts import AemDiagnostic, DiagnosticRequest
from service import DiagnosticUnavailable, diagnose

app = FastAPI()


@app.post("/diagnose")
def create_diagnostic(request: DiagnosticRequest) -> AemDiagnostic:
    try:
        return diagnose(request.context)
    except DiagnosticUnavailable as e:
        raise HTTPException(status_code=502, detail=str(e)) from e
```

**A `BaseModel` parameter means "JSON request body"** — a bare `context: str` parameter
would instead become a *query* parameter. A one-field request model is still worth it:
callers POST `{"context": "..."}`, which is a named, extensible field, and
`Field(min_length=1)` on it rejects an empty context with 422 before any code (or any
token) is spent.

**The return annotation is the response model.** `-> AemDiagnostic` makes FastAPI
validate the object on the way out and publish its schema under
`#/components/schemas/AemDiagnostic` (confirmed in `/openapi.json`). The older explicit
form `@app.post(..., response_model=AemDiagnostic)` still works and *overrides* the
annotation when both are present — which is the documented escape hatch when the handler
returns something wider than it publishes.

## Status codes: three failures, three answers

This is the engineering content of the boundary, not boilerplate.

| Failure | Code | Who writes it |
| --- | --- | --- |
| Client sent `{"context": ""}` or a malformed body | **422** | Nobody — `Field(min_length=1)` produces it |
| Valid request, but the *model* refused, truncated, or returned data failing validation | **502 Bad Gateway** | You: `HTTPException(502)` on the service's own exception |
| Anything unanticipated | **500** | Nobody — leave FastAPI's default alone |

502 is the one people get wrong. The caller did nothing wrong and a retry may well
succeed, so 500 ("my service is broken") misattributes the fault, and returning 200 with
an empty or half-built diagnostic is worse — it pushes the failure downstream disguised
as valid data.

This only stays a one-line `except` because the *service* already funnels both of its
failure modes (`output_parsed is None`, and the `ValidationError` that
`responses.parse()` can raise from inside) into a single application exception. Without
that, the HTTP layer ends up catching Pydantic internals — and a leaked `ValidationError`
becomes a 500 describing the model's misbehavior as a bug in your code.

## Running it

FastAPI gives you `app`: an ASGI callable, not a process. The server is a separate
package.

```zsh
uv add fastapi uvicorn
uv run uvicorn api:app --reload      # run from the directory that holds api.py
```

- `api:app` = import module `api`, take attribute `app`.
- **`--app-dir` defaults to `""`, and uvicorn does `sys.path.insert(0, app_dir)`**
  (`uvicorn/main.py:559`) — i.e. the *current working directory* lands on the import
  path. That is the whole reason flat sibling imports (`from contracts import ...`)
  resolve when you launch from inside the app directory and fail from the repo root.
  Launch from elsewhere with `--app-dir <dir>` rather than rewriting the imports.
- `--reload` is development-only (it runs a file-watching supervisor process).
- `--factory` if `app` is a function returning the app instead of a module-level object.
- `http://127.0.0.1:8000/docs` is Swagger UI; `/openapi.json` is the machine-readable
  contract — this is what a Java consumer generates a DTO/client from, which is the
  point of publishing the schema at all.

## Gotchas

**`def` vs `async def` is a capacity decision, not a style one.** FastAPI checks whether
the endpoint is a coroutine function and routes a plain `def` through
`run_in_threadpool` (`fastapi/routing.py:354`), so a blocking call (the synchronous
OpenAI client, a JDBC-ish driver, `requests`) is safe there. Declare the same handler
`async def` and the blocking call runs *on the event loop* — every other in-flight
request stalls for the whole model round-trip, seconds at a time. The cost of the safe
option is a concurrency ceiling: anyio's default thread limiter is **40** tokens
(measured via `anyio.to_thread.current_default_thread_limiter().total_tokens`), so at
most 40 blocking handlers run at once and the 41st queues. For an LLM endpoint with
multi-second latency, that ceiling is the real throughput number — raise the limiter or
move to the async client before adding workers.

**Field descriptions written for the model become public API documentation.** The same
`Field(description=...)` that steers generation is copied verbatim into `/openapi.json`.
A description phrased at the model ("Your own certainty in this diagnosis, 0.0–1.0.
Lower it when the supplied context is insufficient.") ships to every consumer of the
endpoint — verified in the generated schema. Either phrase descriptions so they read
correctly to both audiences, or keep model-facing guidance in `instructions` and leave
`description` consumer-facing.

**Codes you raise are not codes you document.** The generated OpenAPI lists only
`200` and `422` for the endpoint above — the `502` exists at runtime but appears nowhere
in the contract until you declare it (`@app.post(..., responses={502: {...}})`). A
generated Java client therefore has no branch for the failure that matters most.

**Constraints survive into the published schema, enums included.** `Severity(str, Enum)`
emits `{"type": "string", "enum": ["low","medium","high","critical"]}` and
`Field(ge=0, le=1)` emits `minimum`/`maximum` — so one Pydantic declaration serves three
consumers: it constrains the model's decoding, validates the reply, and documents the
API. Note the asymmetry recorded in the companion doc: the enum is genuinely enforced
during decoding, while `minimum`/`maximum` are only validated client-side after the call.

**The runtime floor is Pydantic v2.** fastapi 0.141.1 declares `Requires-Dist:
pydantic>=2.9.0` and ships only a v2 compat module (`fastapi/_compat/v2.py`); v1-style
models are handled only through the `pydantic.v1` namespace *inside* pydantic 2
(`annotation_is_pydantic_v1` helpers in `_compat/shared.py`), not by a standalone
pydantic 1 install. A codebase still on v1 models migrates or shims — it does not pin.
