# Immutable JSON payload DTOs in Sling servlets: `@Value @Builder @Jacksonized`

## The pattern this applies to

A common AEM connector shape: a servlet accepts a JSON POST body, deserializes it into a DTO
with Jackson, and — because the real work must not run on the request thread — re-serializes
the DTO to a JSON string, puts it in a Sling Job property, and a `JobConsumer` deserializes it
again on the other side:

```
clientlib JS ──POST JSON──▶ SlingAllMethodsServlet
                              MAPPER.readValue(req.getReader(), Context.class)   // deserialize #1
                              jobProps.put("data", MAPPER.writeValueAsString(ctx))
                              jobManager.addJob(TOPIC, jobProps)
                                            │
                              JobConsumer.process(job)
                              MAPPER.readValue(job.getProperty("data"), Context.class)  // deserialize #2
```

The JSON-string-in-a-job-property indirection exists because Sling Job properties only take
simple/serializable types and must survive queue persistence — a JSON string is the robust
choice. Consequence: **every DTO in the payload graph is a Jackson *deserialization* target,
twice.**

## The trap

These DTOs are conceptually immutable value objects (a language `{id, name, locale}`, a path
item, a submission context). The natural Lombok choice is `@Value`. But plain `@Value` removes
the no-arg constructor and setters, and a bare `new ObjectMapper()` (no modules registered —
the normal state in an OSGi bundle) then fails **at runtime only**:

```
InvalidDefinitionException: Cannot construct instance of `...CrowdinLanguage`
(no Creators, like default constructor, exist)
```

Nothing at compile time catches this. So teams "fix" it by keeping `@Getter @Setter` — a
mutable class whose mutability serves nothing but Jackson's default plumbing.

## The fix: `@Jacksonized`

Lombok ≥ 1.18.14 ships `@lombok.extern.jackson.Jacksonized`. Stacked on `@Builder`, it
annotates the generated builder so Jackson deserializes *through the builder* — no modules, no
`lombok.config`, works with a vanilla `ObjectMapper` in both directions (serialization just
uses the getters as always):

```java
@Value
@Builder
@Jacksonized
@JsonIgnoreProperties(ignoreUnknown = true)
public class CrowdinLanguage {
    String id;       // @Value makes fields private final; declare them bare
    String name;
    String locale;
}
```

Under the hood `@Jacksonized` adds `@JsonDeserialize(builder = CrowdinLanguage.CrowdinLanguageBuilder.class)`
on the class and `@JsonPOJOBuilder(withPrefix = "")` on the builder.

### Alternatives, and why not

- `lombok.config` with `lombok.anyConstructor.addConstructorProperties=true` — Jackson honors
  `@java.beans.ConstructorProperties` on the all-args constructor. Works, but it's a global
  config knob affecting every class in the module, and the mechanism is invisible at the class.
- `ParameterNamesModule` / `-parameters` compiler flag — requires registering a module on every
  `ObjectMapper` (easy to miss one; there are usually several mappers scattered across
  servlets/jobs) and a compiler flag.
- `@Jacksonized` is self-contained at the class: whoever reads the DTO sees exactly why it
  deserializes. Prefer it.

## `@JsonIgnoreProperties(ignoreUnknown = true)` — on every class, including nested ones

`@JsonIgnoreProperties` is **not inherited from the containing DTO**. If the top-level context
has it but a nested `CrowdinLanguage` doesn't, one extra key inside a language object fails the
whole request with a 500. And extra keys will happen: the clientlib side of a payload routinely
gains fields before (or independently of) the Java side. Annotate every class in a
client-posted payload graph.

## Converting an existing mutable DTO — checklist

1. Swap `@Getter @Setter` for `@Value @Builder @Jacksonized` + `@JsonIgnoreProperties(ignoreUnknown = true)`;
   declare fields bare (no `private final` — `@Value` adds it).
2. Compile. The setters cease to exist, so **the compiler finds every construction site** —
   convert each to `Type.builder().field(x).build()`. Don't forget test fixtures.
3. Run the unit tests. The Jackson wiring itself fails only at runtime; the tests that
   round-trip the DTO through an `ObjectMapper` (servlet tests, JobConsumer tests) are the only
   thing that actually proves deserialization works.
4. Free win: `@Value` gives real `equals`/`hashCode`, so id-based manual comparisons can go.

## Signature to recognize

`InvalidDefinitionException ... no Creators, like default constructor, exist` from a Sling
servlet or JobConsumer right after someone made a DTO immutable → they used `@Value` (or
removed the no-arg ctor) without `@Jacksonized`.

Confirmed on the Crowdin AEM connector (`CrowdinLanguage` / `SubmitTranslationsContext` flow,
Lombok 1.18.46, AEM SDK 2026.6), July 2026.