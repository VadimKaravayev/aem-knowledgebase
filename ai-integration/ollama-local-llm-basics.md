# Ollama: running LLMs locally — commands, API, gotchas

Ollama runs open-weight models (Llama, Gemma, Qwen, Mistral…) on a laptop behind a local
HTTP server, so the same prompt/contract code written for OpenAI can be exercised offline,
without an API key, and against a model whose weights don't leave the machine (relevant
when the `input` is customer AEM config, logs or content). The server speaks its own
native API **and** an OpenAI-compatible one, including the Responses API.

Verified on Ollama 0.33.3 (macOS menu-bar app, Apple Silicon, 16 GB), `llama3.2:1b`,
openai 3.8.0 / Python 3.13. Items marked *(docs)* are from Ollama's docs and not
re-verified here.

## Mental model

- One **server** (`ollama serve`) owns the models directory, loads models into RAM/VRAM on
  demand, and unloads them after an idle timeout. Everything else — the `ollama` CLI, the
  menu-bar app, your Python — is a client talking to `http://127.0.0.1:11434`.
- On macOS the app in `/Applications/Ollama.app` *is* the server: it spawns
  `Ollama.app/Contents/Resources/ollama serve` at login. Don't run a second `ollama serve`
  in a terminal; it fails to bind the port. `ollama launch` opens the app menu / wires
  Ollama into Claude Code, Codex, VS Code etc.
- A **model** = a manifest + content-addressed blobs (weights, template, params, license)
  in `~/.ollama/models/{manifests,blobs}`. A Modelfile-derived model shares the weight blob
  with its base, so `ollama cp` / `ollama create` cost bytes only for the layers they change.

```bash
curl localhost:11434/            # "Ollama is running"
curl localhost:11434/api/version # {"version":"0.33.3"}
ollama -v
```

## Model management

```bash
ollama pull llama3.2:1b        # tag defaults to :latest; registry = ollama.com/library
ollama list                    # installed models, size, modified
ollama show llama3.2:1b        # arch, params, context length, quantization, capabilities (completion/tools/vision/thinking)
ollama show MODEL --modelfile  # the resolved Modelfile (FROM becomes a blob path)
ollama show MODEL --system | --template | --parameters | --license
ollama ps                      # LOADED models: size in RAM, CPU/GPU split, CONTEXT, UNTIL (unload time)
ollama stop MODEL              # unload now (otherwise unloads after OLLAMA_KEEP_ALIVE, default 5m)
ollama cp MODEL NEW[:tag]      # alias; `ollama rm` accepts several names
ollama rm MODEL ...
ollama push / signin / signout # publish to ollama.com (uses ~/.ollama/id_ed25519)
```

Gotchas verified here:

- **`ollama run <model-not-installed>` auto-pulls it** (`/api/show` 404 → `/api/pull`). A typo
  in a model name becomes a multi-GB download; Ctrl-C leaves `sha256-…-partial*` chunk
  files in `~/.ollama/models/blobs/`. They are resumed by re-running the same pull; to
  discard them, `rm ~/.ollama/models/blobs/sha256-<digest>*-partial*` (nothing references
  them, so this is safe while the server runs). The server also prunes unreferenced blobs on
  startup unless `OLLAMA_NOPRUNE` is set *(docs)*.
- **`ollama ps` CONTEXT ≠ the model's context length.** `show` reported 131072 for
  llama3.2:1b, `ps` showed the loaded instance at **4096**: the server picks a default
  (`OLLAMA_CONTEXT_LENGTH`, "4k/32k/256k based on VRAM") and silently truncates longer
  prompts. Raise it per model (`PARAMETER num_ctx` in a Modelfile), per request
  (`options.num_ctx` on the native API) or globally (env var). Long AEM logs/configs pasted
  into `input` are the canonical victim: the model answers from the *tail* of the prompt.
- Tiny models are fine for plumbing tests and useless for AEM facts: asked for "one OSGi
  annotation", llama3.2:1b answered "Bundle-Scope annotations". Use them to test the
  contract (schema, tool loop, error handling), not the content.

## `ollama run`

```bash
ollama run llama3.2:1b                          # interactive REPL
ollama run llama3.2:1b "one-shot prompt"        # prints answer, exits
cat error.log | ollama run llama3.2:1b "Summarize this AEM error log"   # stdin is appended to the prompt
ollama run llama3.2:1b --format json "Return {city,country} for Lisbon"  # forces valid JSON
ollama run llama3.2:1b --verbose "…"            # prints timings/token rates after the answer
ollama run MODEL --keepalive 30m "…"            # keep loaded longer than the default 5m
ollama run MODEL --think=false / --hidethinking # thinking models only
```

- REPL slash commands *(docs)*: `/set system "…"`, `/set parameter num_ctx 8192`,
  `/show info|modelfile|parameters`, `/save NAME` (persist the current system/params as a
  new model), `/load NAME`, `/clear`, `/bye`, `/?`. Multi-line input is wrapped in `"""`.
  Prompts typed in the REPL are kept in plain text in `~/.ollama/history`.
- **Non-interactive stdin trap:** when stdin is not a TTY, `ollama run MODEL "prompt"` reads
  stdin **to EOF** and appends it to the prompt *before* sending anything. `sleep 4 |
  ollama run … "prompt"` took 4.1 s vs 1.8 s with `</dev/null`. From a background job, a
  cron entry, or an agent harness whose stdin never closes it **hangs forever** with no
  output. Always `</dev/null` in scripts. Corollary: `printf '/?\n/bye\n' | ollama run …`
  does not run slash commands — they are sent to the model as text.

## Native HTTP API (`/api/*`)

Streaming NDJSON is the default; send `"stream": false` for one JSON object.

```bash
curl localhost:11434/api/generate -d '{"model":"llama3.2:1b","prompt":"Say hi","stream":false}'
# → {"response":"Hello.","done":true,"done_reason":"stop","eval_count":3,"total_duration":…ns}

curl localhost:11434/api/chat -d '{"model":"llama3.2:1b","stream":false,
  "messages":[{"role":"system","content":"You are terse."},{"role":"user","content":"…"}],
  "options":{"temperature":0.2,"num_ctx":8192}}'
# → {"message":{"role":"assistant","content":"…"},…}
```

- **Structured output:** `"format": "json"` (any JSON) or `"format": {<JSON schema>}` —
  the schema form is constrained decoding; verified it returned exactly
  `{"city":"Lisbon","country":"Portugal"}` with no prose. Same caveat as
  [llm-structured-output-contracts.md](llm-structured-output-contracts.md): structure is
  enforced, semantics are not.
- **Tool calling** on `/api/chat` with `"tools":[{"type":"function","function":{…}}]` for
  models whose `show` lists `tools`: the reply is
  `{"message":{"content":"","tool_calls":[{"id":"call_…","function":{"name":"get_weather","arguments":{"city":"Lisbon"}}}]}}`.
  Note **`arguments` is already an object**, not the JSON-encoded string OpenAI returns
  (compare [openai-function-calling-responses.md](openai-function-calling-responses.md)) —
  code that `json.loads` it breaks. You execute the function and append a
  `{"role":"tool","content":"…"}` message, then call again.
- Other endpoints: `GET /api/tags` (= `list`), `GET /api/ps`, `POST /api/show`,
  `POST /api/pull`, `POST /api/embed` (`"input"` string or list), `POST /api/create`,
  `DELETE /api/delete`.
- Per-request `keep_alive` ("0" unloads right after the call, "-1" pins the model).

## OpenAI-compatible API (`/v1/*`) — point existing SDK code at Ollama

```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")  # key is required by the SDK, ignored by Ollama

r = client.responses.create(model="llama3.2:1b", instructions="You are terse.", input="Capital of Portugal?")
r.output_text            # "Lisbon."  — same instructions/input/output_text shape as OpenAI
r.usage                  # input/output/total tokens incl. cached_tokens

class Place(BaseModel): city: str; country: str
client.responses.parse(model="llama3.2:1b", input="Lisbon: city and country.", text_format=Place).output_parsed
# Place(city='Lisbon', country='Portugal')

client.responses.create(model="llama3.2:1b", input="hi", previous_response_id=r.id)   # chaining works
```

Verified on 0.33.3: `GET /v1/models`, `POST /v1/chat/completions`, `POST /v1/responses`
(with `instructions`, `input`, `text.format` json_schema via `responses.parse`,
`previous_response_id`). So Week-1..3 style code that keeps `instructions` and `input`
separate runs unchanged offline by swapping `base_url` and `model`. Not verified: tool
calling and streaming over `/v1/responses`; use `/v1/chat/completions` or the native API
if those misbehave. The reply reports `"store": false` but `previous_response_id` still
resolved within the server's lifetime — don't rely on it surviving a restart.

## Modelfile: bake a system prompt and params into a named model

```dockerfile
# Modelfile
FROM llama3.2:1b
SYSTEM """You are an AEM (Adobe Experience Manager) developer assistant. Answer tersely."""
PARAMETER temperature 0.2
PARAMETER num_ctx 8192
```

```bash
ollama create aem-helper -f Modelfile     # "creating new layer … writing manifest … success"; ~seconds, no weight copy
ollama run aem-helper "What does @Component do?"
ollama show aem-helper --modelfile        # FROM is rewritten to the base's blob path
```

`TEMPLATE` (Go template that renders messages/tools into the model's chat format) is
inherited from the base; only override it when importing raw GGUF/safetensors (`FROM
./model.gguf`, `--quantize q4_K_M`). Baking the system prompt into a model is handy for
the REPL and for tools that only take a model name; for application code keep the system
prompt in `instructions` in the request so it's versioned with the code.

## Environment variables that matter (set on the *server*)

| Var | Default | Why you'd set it |
|---|---|---|
| `OLLAMA_HOST` | `127.0.0.1:11434` | `0.0.0.0:11434` to reach the server from a VM/container/other machine; the CLI reads the same var to find the server |
| `OLLAMA_MODELS` | `~/.ollama/models` | move multi-GB weights to another disk |
| `OLLAMA_KEEP_ALIVE` | `5m` | `-1` keep loaded, `0` unload immediately; the first request after unload pays the load time (19 s for a 270M model cold on a laptop) |
| `OLLAMA_CONTEXT_LENGTH` | 4k/32k/256k by VRAM | see the `ps` CONTEXT trap above |
| `OLLAMA_NUM_PARALLEL` / `OLLAMA_MAX_LOADED_MODELS` / `OLLAMA_MAX_QUEUE` | auto | concurrency; parallel slots multiply KV-cache memory |
| `OLLAMA_ORIGINS` | localhost | CORS allow-list when a browser page calls the API directly |
| `OLLAMA_NO_CLOUD` | unset | disable ollama.com remote inference / web search features |
| `OLLAMA_DEBUG=1` | | verbose server log |

For the macOS app these must be set where launchd sees them —
`launchctl setenv OLLAMA_HOST "0.0.0.0:11434"` then quit and relaunch the app *(docs)*; a
terminal `export` only affects CLI clients started from that shell.

## Where things live (macOS)

- `~/.ollama/models/blobs/` and `manifests/registry.ollama.ai/library/<model>/<tag>` — the
  `-partial*` files are interrupted pulls.
- `~/.ollama/logs/server.log` — every request as a GIN access-log line
  (`POST "/api/chat" | 200 | 215ms`), model load decisions ("model fits alongside existing
  models", VRAM estimates), and the full llama-server startup. First place to look when a
  request is slow (watch for a load) or a pull appears from nowhere (`/api/show` 404 followed
  by `/api/pull`).
- `~/.ollama/history` — plain-text REPL prompt history. `~/.ollama/id_ed25519` — the key
  used for `push`/`signin`.
