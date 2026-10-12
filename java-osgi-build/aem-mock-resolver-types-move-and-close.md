# AEM Mock: `RESOURCERESOLVER_MOCK` cannot `move`, `JCR_MOCK` really closes the resolver

## Symptom

A repository service that archives a record with `resolver.move(src, destParent)` passes in AEM but its
unit test on the default `RESOURCERESOLVER_MOCK` context fails with a bare
`java.lang.UnsupportedOperationException` from the move. Switching the whole test class to
`new AemContext(ResourceResolverType.JCR_MOCK)` makes every *other* test in the class fail with
`IllegalStateException: Resource resolver is already closed.`

## Root cause

The two Sling mocks differ in two places that matter for repository code:

| | `RESOURCERESOLVER_MOCK` | `JCR_MOCK` |
|---|---|---|
| `ResourceResolver.move` | throws `UnsupportedOperationException` | implemented (`Session.move`) |
| `ResourceResolver.close()` | no-op | really closes; later calls throw |
| transient space | yes, `commit()` matters | none, writes visible at once |

Production code written as `try (ResourceResolver r = provider.getServiceResolver()) { … }` closes the
resolver on every call. A test that stubs the provider with `context.resourceResolver()` hands the
context's one resolver to every call, which only works because the default mock ignores `close()`.

## Fix

Keep the existing class on `RESOURCERESOLVER_MOCK` and put the move tests in their own class on
`JCR_MOCK`, stubbing the provider to hand out a **fresh resolver per call**, the way production does:

```java
private final AemContext context = new AemContext(ResourceResolverType.JCR_MOCK);

@BeforeEach
void setUp() throws LoginException {
    ResourceResolverFactory resolvers = context.getService(ResourceResolverFactory.class);
    when(resolverProvider.getServiceResolver())
            .thenAnswer(invocation -> resolvers.getAdministrativeResourceResolver(null));
    …
}
```

Assertions read through `context.resourceResolver()`; with no transient space in `JCR_MOCK`, writes
committed by the per-call resolvers are visible there immediately.

Do **not** add a copy-and-delete fallback on `UnsupportedOperationException` in production to make
the default mock pass (the Crowdin AEM connector's `moveTranslationRequestToArchive` does this): `/var`
is JCR-backed in every AEM, so that branch only ever runs in tests.

## How to spot it next time

- `UnsupportedOperationException` with no message, from a `move` or `copy`, under `MockResourceResolver` in the stack.
- "Resource resolver is already closed" from a class that passes `context.resourceResolver()` into a
  service using try-with-resources, right after switching to `JCR_MOCK`.
- There is no `context.resourceResolverFactory()` accessor; get the factory with
  `context.getService(ResourceResolverFactory.class)`.
