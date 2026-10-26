# OpenAI function calling (Responses API) — the tool-loop contract

What the OpenAI function-calling guide actually specifies, Responses-API mode, for
building an agentic assistant on the raw SDK (no LangChain/MCP): how a tool is
declared, what the model sends back, what you must echo, and which defaults differ
silently from Chat Completions. Source:
<https://developers.openai.com/api/docs/guides/function-calling?api-mode=responses>.

Companion to [[llm-structured-output-contracts]] (same constrained-decoding machinery —
strict function arguments ARE structured outputs under the hood) and
[[fastapi-llm-service-boundary]] (where a validated result goes next).

---

## The loop

Five steps, repeated until the model stops asking:

1. request with `tools` defined →
2. response whose `output` array contains `function_call` items →
3. **you** execute the matching code →
4. second request carrying `function_call_output` items →
5. final text response (or more tool calls — loop).

Nothing executes server-side; a "tool call" is just the model emitting a typed
request that your application chooses to honor. The loop is plain HTTP + JSON —
no framework needed.

---

## Declaring a function — and the shape trap

Responses-mode tool definitions are **flat**; Chat Completions nests the same
fields under a `function:` key. Copy-pasting a Chat Completions definition into
`responses.create` (or vice versa) is the classic silent-ish failure when porting
examples:

```json
{
  "type": "function",
  "name": "get_weather",
  "description": "Retrieves current weather for the given location.",
  "parameters": {
    "type": "object",
    "properties": {
      "location": { "type": "string", "description": "City and country e.g. Bogotá, Colombia" },
      "units": { "type": "string", "enum": ["celsius", "fahrenheit"] }
    },
    "required": ["location", "units"],
    "additionalProperties": false
  },
  "strict": true
}
```

`parameters` is ordinary JSON Schema (types, enums, descriptions, nested and even
recursive objects). SDK helpers exist (`pydantic_function_tool(Model)` in Python,
`zodFunction` in JS) but **don't support every schema feature** — OpenAI's own
guidance is to write the schema directly when possible.

---

## What comes back, what you send back

A function call arrives as an item in `response.output` (never assume exactly one —
**zero, one, or many per response**, including several calls to the *same* function):

```json
{ "id": "fc_12345xyz", "call_id": "call_12345xyz", "type": "function_call",
  "name": "get_weather", "arguments": "{\"location\":\"Paris, France\"}" }
```

- **Two ids.** `id` is the output-item id (`fc_…`); `call_id` (`call_…`) is what you
  must echo back. Pairing the result with the wrong one is an easy bug because both
  look plausible.
- **`arguments` is a JSON-encoded *string*** — `json.loads` it; it is not an object.
- The result goes back as an input item:

```json
{ "type": "function_call_output", "call_id": "call_12345xyz", "output": "..." }
```

`output` is typically a string whose format is up to you (JSON, plain text, error
codes — the model interprets it); for image/file-returning tools it can be an array
of image/file objects; for void functions return `"success"`/failure explicitly.
Because each output references its `call_id`, **results can be returned in any
order**.

---

## Carrying state between the two requests

Two mechanisms, both from the guide:

1. **Running input list:** `input_list += response.output`, then append your
   `function_call_output` items, then call again with the whole list. You must
   append the **entire** model output, not just the function calls — **for reasoning
   models, the reasoning items that accompanied the tool calls must be passed back
   with the outputs** or the follow-up request degrades/fails.
2. **`previous_response_id`:** send only the new `function_call_output` items as
   `input` and point `previous_response_id` at the prior response — the server
   reconstructs the history (including reasoning items) for you.

Either way, keep sending the same `tools` list on the follow-up call.

---

## Strict mode — the defaults diverge between APIs

`strict: true` routes argument generation through structured outputs: same
requirements as [[llm-structured-output-contracts]] — `additionalProperties: false`
on every object, **every** property listed in `required`, optional fields expressed
as `"type": ["string", "null"]`. Same guarantees and the same asymmetry too:
structure and enums are decoder-enforced, range/length keywords are not.

The part worth memorizing because it's silent:

- **Explicit `strict: true` + non-conforming schema → request rejected** with
  details (good: loud).
- **`strict` omitted on Responses → the API tries to *normalize* your schema into
  strict mode**, and if it can't, falls back to non-strict best-effort calling —
  the only tell is `strict: false` on the tool echoed in the response. So "I never
  set strict and it mostly works" means you may be strict on some tools and
  best-effort on others without noticing.
- **`strict` omitted on Chat Completions → always non-strict.** Different default,
  same parameter name.
- Opting out deliberately on Responses requires explicit `strict: false`.

Fine-tuned-model caveats: schemas are preprocessed on first request then cached
(varying schemas per request = latency; cached schemas are **not eligible for zero
data retention**), and if a fine-tuned model emits multiple parallel calls in one
turn, **strict mode is disabled for those calls**.

---

## tool_choice and parallel calls

- `"auto"` (default): zero, one, or many calls. `"required"`: at least one.
  `{"type": "function", "name": "get_weather"}`: force that one. `"none"`: behave
  as if no tools were passed.
- `allowed_tools` restricts the callable subset **without changing the `tools`
  list**, which is exactly the point: the full stable tool list keeps the prompt
  cache warm while you vary what's callable per request:

```json
"tool_choice": { "type": "allowed_tools", "mode": "auto",
  "tools": [ { "type": "function", "name": "get_weather" } ] }
```

- `parallel_tool_calls: false` guarantees **zero or one** call per turn — the knob
  to reach for when a sequential assistant mustn't fan out. Built-in tools
  (web search etc.) are never included in a parallel function-call batch.

---

## Designing the tool surface (guide's best practices, condensed)

- **The intern test:** could a human correctly use the function given *only* what
  the model sees (name, description, schema, system prompt)? Every question the
  intern would ask you is a sentence missing from the prompt.
- Names/descriptions do the work; put *when-and-when-not-to-use* policy in the
  system prompt (`instructions`), field-level format rules in parameter
  descriptions — same three-channel split as [[llm-structured-output-contracts]].
- **Use enums/structure to make invalid states unrepresentable** — the guide's own
  example of a bad schema: `toggle_light(on: bool, off: bool)`.
- **Don't make the model fill arguments you already know** (have `submit_refund()`
  take no `order_id` if your code already holds it), and merge functions that are
  always called in sequence. Fewer, dumber decisions = higher accuracy.
- Examples in the prompt fix recurring failures — but **may hurt reasoning models**.
- **Soft ceiling ~20 functions callable at the start of a turn**; definitions are
  injected into the system message, so every tool costs input tokens on every
  request.

---

## Streaming the Responses way

Per function call: `response.output_item.added` (carries the item with `name`,
`call_id`, empty `arguments`) → N × `response.function_call_arguments.delta`
(accumulate `delta` keyed by `output_index`) → `response.function_call_arguments.done`
(full `arguments`) → `response.output_item.done`. You aggregate an *arguments JSON
string*, not display text — nothing is parseable until `done`. (Chat Completions
streams the same information as `delta.tool_calls` chunks indexed by `index`, with
`id`/`name`/`type` present only on each call's first chunk.)

---

## Adjacent features (same page, newer models — noted, not yet load-bearing)

- **Namespaces:** `{"type": "namespace", "name": "crm", "tools": [...]}` groups
  tools by domain; member functions can set `defer_loading: true`.
- **Tool search:** `tool_search` lets the model discover and load deferred tools
  mid-conversation (gpt-5.4+ only); `tool_search_call`/`tool_search_output` items
  precede the resulting `function_call`, which you handle normally. `tool_choice`
  still applies to whatever is currently loaded.
- **Custom tools:** `{"type": "custom", "name": "code_exec", "description": ...}` —
  no schema; the model passes a free-form string, arriving as a `custom_tool_call`
  item whose payload field is **`input`** (not `arguments`). Optionally constrained
  by a context-free grammar: `format: {"type": "grammar", "syntax": "lark"|"regex",
  "definition": ...}`. The right shape when the natural payload is code or DSL text
  and JSON-wrapping would just add escaping.

Docs-based (page used `gpt-6-astra` for Responses examples, `gpt-5.6-terra`
for Chat Completions; GPT-6-generation models are Responses-only for tool calling).
Read from the live page, not reproduced against the API.
