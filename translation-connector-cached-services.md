# Translation-connector cached services: rules service & cloud-config service

Every AEM translation connector grows the same two JCR-backed, read-mostly, hot-path services:

| | reads | consumers |
|---|---|---|
**rules service** | `translation_rules.xml` (`/conf/global` → `/apps` → `/etc` → `/libs`) | every node/property translatability check, the rules report, XLIFF serialization |
**cloud-config service** | credentials at `/conf/<vendor>/settings/config` (token via `CryptoSupport`) | almost every operation — API client building, storage resolution |

Both must be cached (a service-user login + JCR walk + XML parse, or a login + decrypt, per property
check is not viable), and caching them is where connectors get it wrong. This is the contract that
works on AEM as a Cloud Service.

## The cache needs three mechanisms, not one

Each closes a hole the other two cannot:

1. **`ExternalResourceChangeListener` marker** — cross-pod delivery. Without it a change event only ever
   reaches the pod that persisted the write.
2. **A version stamp sampled before the read** — an event that arrives *during* a read still takes effect.
3. **A TTL backstop** — an event that is never delivered at all still cannot pin a stale value.

Shipping only (1) leaves a lost-invalidation race on a single pod. Shipping only (3) means minutes of
stale rules, which for a mid-flight submit means a corrupt job, not a slow one. Shipping only (2) is
useless without (1).

### 1. The marker (the AEMaaCS-specific bug)

AEMaaCS author is "a cluster of AEM author pods sharing a single content repository" with a minimum of
two pods. Plain `ResourceChangeListener` "gets only local events which means events caused by changes
persisted on the same instance as this listener is registered" (Sling javadoc). So a rules edit or a
token rotation clears the cache on one pod; the others serve pre-change data until they recycle — which
is why the fault looks self-healing and never reproduces on a local Quickstart or a single-node on-prem
instance.

```java
@Component(
        service = {TranslationRulesService.class, ResourceChangeListener.class},   // ← unchanged
        property = {
                ResourceChangeListener.PATHS + "=" + CONF_RULES_FILE_PATH,
                ResourceChangeListener.CHANGES + "=ADDED",
                ResourceChangeListener.CHANGES + "=CHANGED",
                ResourceChangeListener.CHANGES + "=REMOVED"
        })
public class DefaultTranslationRulesService
        implements TranslationRulesService, ResourceChangeListener, ExternalResourceChangeListener {
```

Registration traps:

- **`ExternalResourceChangeListener` does NOT extend `ResourceChangeListener`** — it is an empty marker
  (`org.apache.sling.api.resource.observation`, Sling API 2.11.0+). Implement **both**, and keep
  `ResourceChangeListener.class` in the `service` list: *"the implementation should implement the
  `ExternalResourceChangeListener` interface, but still register the service as a
  `ResourceChangeListener` service."* Registering under the marker interface instead deregisters the
  listener entirely — silently.
- **`resource.paths` is required.** A listener component that declares no `PATHS` is ignored by Sling
  outright, with no warning. (Typical victim: the archetype's leftover `SimpleResourceListener`.)
- **`PATHS` is subtree-inclusive**, so watching the `.xml` node catches its `jcr:content/jcr:data` write.
  Path registration is rarely the bug; missing external delivery is.

### 2. Version stamp — the mid-read race

The tempting shape is a lazy `AtomicReference` with `updateAndGet`, invalidated by `set(null)`. It loses
events:

```
T1: cache empty → readTranslationRules() starts (login + JCR walk + parse, tens of ms)
T2: author saves the file → onChange → cache.set(null)   ← lands on an already-empty cache, evaporates
T3: T1 finishes holding pre-save content → cache.set(old)
T4: cache now serves stale rules with no pending invalidation left to correct it
```

The invalidation was expressed as an *absence*, and a racing reader filled the absence back in. Express
it as a monotonic fact instead, sampled **before** the load:

```java
private final AtomicReference<CachedRules> cache = new AtomicReference<>();
/** Bumped on every observed change; identifies which version of the file a cached value came from. */
private final AtomicLong version = new AtomicLong();

@Override
public void onChange(List<ResourceChange> changes) {
    version.incrementAndGet();
    cache.set(null);            // optional: releases the stale object now rather than at the next read
}

@Override
public TranslationRules getTranslationRules() {
    long currentVersion = version.get();          // sample BEFORE the read — this is the whole trick
    CachedRules cached = cache.get();
    if (cached != null
            && cached.getReadAtVersion() == currentVersion
            && (now() - cached.getReadAtMs()) < CACHE_TTL_MS) {
        return cached.getRules();
    }
    TranslationRules fresh = readTranslationRules();
    cache.set(new CachedRules(fresh, currentVersion, now()));
    return fresh;
}

@Value
private static class CachedRules {
    TranslationRules rules;
    long readAtVersion;   // parallel to readAtMs: "read at what version" / "read at what time"
    long readAtMs;
}
```

A value parsed across a concurrent `onChange` carries the superseded version, so the next caller misses
it — the change cannot be swallowed. Nothing needs to be atomic across the whole read, and two racing
readers can only cost an extra load, never correctness. **No `synchronized`, no `Lock`** — Cloud Manager
flags every monitor use (AEM-15), and a `Lock` here is the same blocking semantics wearing a hat.

### 3. TTL backstop

60 s is the right order of magnitude (matches a credentials cache; a token rotation or rule edit
propagating in under a minute is indistinguishable from instant to an author). Add a package-private
`long now()` seam so tests never sleep.

Decide deliberately whether the TTL applies to a *negative* result:

- **Credentials:** never cache the failure fallback. A transient login/decrypt failure must be retried,
  not pinned for the TTL — `if (fresh != EMPTY) cache.set(...)`.
- **Rules:** "no rules file exists" is a legitimate, stable answer and is worth caching, but note the
  cost the TTL introduces — a `nodeExists` sweep of all four priority paths every 60 s per pod on every
  install that has no rules file.

## Beyond the cache

- **Fetch the ruleset once per pass and thread it through the traversal.** Never call the service
  per node or per property: even a correct cache can flip mid-walk on a TTL boundary, and a single
  submit that used two rulesets is unexplainable in a support ticket. One snapshot per report render and
  one per XLIFF serialization.
- **Report vs export divergence is the canonical field symptom** of any per-pod cache here: the rules
  report is a routed HTTP request, XLIFF serialization usually runs in a Sling job on whichever pod
  picked it up. Two pods, two caches, one confused user ("the report says it's translatable").
- **Evaluate on the JCR API** (`Node`/`Property`), not Sling resources — rule lookup is per-property hot.
- **If the rules lookup ever becomes context-aware** (`/conf/<tenant>` via caconfig instead of a fixed
  `/conf/global`), the cache becomes a `ConcurrentHashMap` keyed by resolved conf root, and `onChange`
  must bump the *global* version rather than clearing one key — the changed file's path does not tell you
  which tenants inherited it.
- **Watch the two-call reads** in the loader itself: `session.nodeExists(p)` followed by
  `session.getNode(p)` doubles the repository round-trips, and with a TTL that now runs every minute.

## Testing recipe (what actually catches these)

- **aem-mock rejects a subclass.** `context.registerInjectActivateService(...)` resolves SCR metadata by
  class name, so an anonymous subclass overriding `now()` fails with
  `NoScrMetadataException: No OSGi SCR metadata found for class …Test$1`. Keep the registered real
  component for the normal tests, and hand-wire the time-controlled instance for TTL tests:
  `FieldUtils.writeField(instance, "serviceUserResolverProvider", provider, true)`. (The pure-Mockito
  sibling test can just use `@Spy @InjectMocks` and stub `now()`.)
- **Test the mid-read race deterministically** — no threads needed. Make the resolver provider fire the
  invalidation on its first invocation, i.e. while the read is in flight:

  ```java
  AtomicBoolean once = new AtomicBoolean(true);
  when(provider.getCrowdinReadOnlyResolver()).thenAnswer(inv -> {
      if (once.getAndSet(false)) service.onChange(Collections.emptyList());
      return resolver;
  });
  service.getTranslationRules();   // stamped with the superseded version
  service.getTranslationRules();   // must re-load
  service.getTranslationRules();   // now a cache hit
  verify(provider, times(2)).getCrowdinReadOnlyResolver();
  ```

  Then **prove the test discriminates**: temporarily drop the version clause from the cache condition and
  watch it fail. A TTL-only implementation passes a badly written version of this test.
- **Guard the marker with a test.** `assertTrue(ExternalResourceChangeListener.class.isAssignableFrom(
  DefaultTranslationRulesService.class))` — nothing else can catch its removal. The compiler is happy
  without it, every unit test passes, and the failure only appears on a multi-pod cloud environment.

## Prior art (this is a repeat offense)

The translated.com connector shipped the same rules cache, then *removed* its 1-minute TTL in favor of
listener-only invalidation, and later had to re-add a 5-minute TTL on a sibling service after the same
failure — without back-porting it to the rules service. Neither that connector nor Crowdin's ever had the
marker. Treat "invalidated by a `ResourceChangeListener`" as an incomplete cache on AEMaaCS.

---

Confirmed against the AEM SDK sources (`aem-sdk-api` 2026.6, `ResourceChangeListener` /
`ExternalResourceChangeListener` javadoc) and Adobe's AEMaaCS architecture documentation; implemented and
unit-tested on the Crowdin connector, September 2026. Related:
[translation-rules-xml.md](translation-rules-xml.md),
[immutable-json-dtos-lombok-jacksonized.md](immutable-json-dtos-lombok-jacksonized.md),
[aem-mock-delegation-picker-trap.md](aem-mock-delegation-picker-trap.md).