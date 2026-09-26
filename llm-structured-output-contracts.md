# LLM structured output — how the model decides what goes in each field

For any AEM-adjacent assistant that must hand a **validated object** to downstream
Java/AEM code rather than a paragraph of prose. Worked out while building an AEM
diagnostic service (OpenAI Responses API + Pydantic), but the mechanics are the
model-boundary ones, not framework trivia.

The question this answers: you declare a schema, the model returns data that fits
it — *what actually tells the model what each field means, and which parts of your
schema are guaranteed versus merely requested?*

---

## The form model

Don't picture asking the model for an essay. Picture handing it a **form** with a
fixed set of boxes, plus one rule: the only thing you may hand back is this form,
filled in.

Three things decide what lands in each box, and they are worth keeping distinct
because only the first is mechanically enforced.

1. **The box labels** — a box literally named `summary` or `likely_cause`. This is
   most of the answer. Nothing explains what "summary" means; the model has seen
   enough forms to know what a box with that name wants. Name a box `xyz` and you
   get garbage. **Field names are the specification.**
2. **The small print under a box** — a per-field `description`. It travels inside
   the schema and is read at the moment that field is generated, which is why it is
   the right place for field-local rules ("quote verbatim from the supplied context,
   do not paraphrase") rather than burying them in a global preamble.
3. **The covering letter and the evidence pile** — `instructions` carries
   cross-field policy ("judge severity by impact on the running instance", "name the
   single most probable cause"); `input` carries the context the values must be
   drawn from.

Schema says *what shape*. Descriptions say *how to fill this box*. Instructions say
*how to decide*. Input says *from what*.

---

## Some boxes are locked; others are only a polite request

This is the part that bites. A Pydantic model is converted to JSON Schema and sent
with `strict: true`, and constrained decoding masks out tokens that would break the
**structure**. Enforced, i.e. cannot come back wrong:

- every declared key present (`required` lists all of them), and
  `additionalProperties: false` — no bonus fields;
- declared types (`string`, `number`, `array` of `string`);
- **`enum` members.** A `Severity(str, Enum)` field cannot return `"sev-1"`,
  `"High"`, or `"super bad"`. Those tokens are unavailable, not merely discouraged.

Not enforced by the decoder, despite appearing in the payload: the **range and
length keywords**. A model with `Field(ge=0.0, le=1.0)` and
`Field(min_length=1)` emits this (dumped via
`openai.lib._pydantic.to_strict_json_schema`):

```json
"evidence":   { "type": "array", "items": {"type":"string"}, "minItems": 1 },
"confidence": { "type": "number", "minimum": 0.0, "maximum": 1.0 }
```

`minItems` / `minimum` / `maximum` are sent, but strict mode enforces only a
limited keyword subset — treat these as *hints to the model*, not guarantees.
What actually enforces them is **Pydantic, client-side, after the JSON returns**:

```
2 validation errors for AemDiagnostic
evidence    List should have at least 1 item after validation, not 0 [type=too_short]
confidence  Input should be less than or equal to 1 [type=less_than_equal]
```

**Consequence for the service boundary:** `client.responses.parse()` can raise a
`ValidationError` *from inside* when the model returns an out-of-constraint value.
A `diagnose()` wrapper that only guards the `None` case is incomplete — catch the
`ValidationError` too and re-raise it as your own domain exception, so callers see
one failure type instead of a Pydantic internal.

Rule of thumb: **put the guarantees you actually depend on in enums and types**
(structure), and treat every numeric range or list-length rule as something your
code must re-check, not something the model was prevented from breaking.

---

## Field order is generation order

The model emits keys in schema order, and each field it has written becomes context
for the next. Declaring `summary`, `likely_cause`, `evidence` *before* `confidence`
means that by the time it scores confidence it can see the cause and the citations
it already committed to. Put `confidence` first and it is guessing its own certainty
before doing any reasoning.

Ordering a contract so conclusions precede self-assessment is a real design lever,
not cosmetics.

---

## Trap: a required non-empty list forces fabrication

Under constrained decoding the model *must* fill every required box. So a
`suggested_actions: list[str]` declared with `min_length=1` tells the model: invent
a remediation step even when the supplied context supports none. That is a
schema-shaped instruction to hallucinate.

The asymmetry worth copying:

- `evidence` **keeps** `min_length=1` — the caller always supplies the diagnostic
  context, so at least one real citation is always available. An empty `evidence`
  means the answer is ungrounded and deserves to fail validation.
- `suggested_actions` is left **unconstrained** — an empty list is a legitimate,
  honest answer, and the consumer renders it as "none" rather than printing an
  empty heading.

Corollary for consumers: handle the empty list **explicitly**. Iterating an empty
list silently prints nothing, which reads as a broken UI rather than as "no action
supported."

---

## Valid ≠ correct

Everything above buys **shape**, and nothing above buys **truth**. A response can
satisfy the schema perfectly, land inside every range, use a real enum member — and
still name the wrong bundle, cite a log line that means something else, or report
`confidence: 0.95` on a guess. `confidence` is an application-facing signal the
model asserts about itself, not a calibrated probability; branch on it if useful,
but do not treat it as measured.

Schema validation moves the failure mode from *unparseable* to *parseable and
possibly wrong* — which is strictly better, because now the wrongness is the only
thing left to review.

---

## Practical notes

- **`output_parsed` is `Optional`.** `ParsedResponse.output_parsed` is typed
  `T | None`, so an IDE will offer to widen your `-> AemDiagnostic` return type to
  `| None`. Don't: the right fix is a `None` check that raises your own exception,
  so callers get a value or an error, never a maybe.
- **Dump the schema before debugging a prompt.** `to_strict_json_schema(Model)`
  shows exactly what the model was handed, descriptions included. A field that comes
  back consistently wrong is usually a field with no `description`.
- **Cheapest experiment** that proves the description channel is live: add
  `description="One sentence, max 15 words, naming the affected component"` to a
  bare field and re-run.

Verified locally Sep 2026 on openai 3.8.0 / pydantic 2.13.5 / Python 3.13: schema
payload dumped as shown, and the two-error `ValidationError` reproduced by feeding
`model_validate` an empty `evidence` plus `confidence: 1.4`. The strict-mode keyword
subset is the vendor-side claim here — the payload contents are confirmed, the
decoder's treatment of `minItems`/`minimum` is not something a local run can prove,
so the client-side guard is the load-bearing part either way.
