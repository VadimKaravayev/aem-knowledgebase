# URL resolution on AEMaaCS: which layer owns which rule

A public URL is not the JCR path. It is produced by several layers working
together, and a rule that sits in the wrong layer usually fails quietly: links
come out wrong, or the cache stops being invalidated. This doc covers the
layering and how to pick a mechanism. The Sling side in detail (the `/etc/map`
tree, alias and vanity properties, the reverse-mapping trap, the
authentication hole) is in
[sling-resource-resolution-mapping.md](sling-resource-resolution-mapping.md).

**Sourcing:** Adobe Experience League Community article *Understanding URL
Resolution in AEM as a Cloud Service: Sling Mappings, Aliases, Vanity URLs and
Dispatcher* (community author, 22 Sept 2026), read September 2026. This is a
community post, not Adobe product documentation. Where it quotes "Adobe
guidance", that is the article's attribution; the underlying Adobe pages were
not checked here. Nothing here was reproduced on a running instance. My own
inferences are flagged.

---

## Two directions, two tables

| Direction | API | Sling table | Example |
|---|---|---|---|
| Incoming | `ResourceResolver.resolve()` | Resolver Map Entries | `/es/cursos/java` → `/content/acme-learning/es/courses/java` |
| Outgoing | `ResourceResolver.map()` | Mapping Map Entries | `/content/acme-learning/es/courses/java` → `/es/cursos/java` |

The test that matters: **`resolve(map(resource))` must return the same
resource.** If only `resolve()` is tested, you catch just half the bugs.

Also keep redirect and rewrite apart. *"A redirect changes the URL the client
uses. A rewrite changes how the server resolves that URL."*

## The incoming chain

```
Browser → CDN → Apache (mod_rewrite) → Dispatcher → Sling Resource Resolver → resource → render
                                                     (mappings, aliases, vanity paths)
```

### Apache: normalise, don't route

```apache
RewriteRule ^/(en|es|fr)/(.*)$ /content/acme-learning/$1/$2 [PT,L]
```

`PT` (pass-through) hands the rewritten URI on to the Dispatcher handler, so
the **Dispatcher caches under the internal `/content/...` path**. The article's
principle is that Dispatcher should *"normalize HTTP requests, not become a
second content repository containing hundreds of business-specific routing
rules."* A rule that depends on business content (localised slugs, campaign
URLs) belongs in the content: in an alias or a vanity path, not in a rewrite
file that needs a pipeline run to change.

### The invalidation mismatch

```
Cached by Dispatcher:          /es/courses/java.html
Invalidation sent by AEM:      /content/acme-learning/es/courses/java
Result:                        stale page, no error
```

AEM invalidates by repository path. If the cache is keyed on the public path,
flushes don't match and nothing reports it. Rewriting to the internal path
*before* Dispatcher (the `PT` rule above) keeps the cache key and the flush
path the same.

*(Inference:)* anything that makes the cached path differ from the
flushed path has the same failure. That includes localised segments served
through `sling:alias` with no Apache rewrite to the node name. Check which path
actually lands in the docroot before you trust auto-invalidation.

### Dispatcher vanity URLs

```
/vanity_urls {
  /url   "/libs/granite/dispatcher/content/vanityUrls.html"
  /file  "/tmp/vanity_urls"
  /delay 300
}
```

Dispatcher polls AEM for the list of published vanity paths, stores it locally,
and **lets a request through that its filters would otherwise deny** when the
request matches a published vanity path. Without this block, a strict
allowlist `/filter` blocks vanity URLs before Sling ever sees them. `/delay` is
in seconds, so a newly published vanity path can take up to about 5 minutes to
work at the edge. *(The article doesn't discuss whether the `/url` endpoint
itself needs a filter allow on AEMaaCS. Unverified.)*

## Sling: alias vs vanity path

*"An alias changes how a resource can be addressed inside its hierarchy. A
vanity URL creates an additional shortcut."*

| | `sling:alias` | `sling:vanityPath` |
|---|---|---|
| Scope | Renames one path segment in place | Adds a whole separate path |
| Direction | Incoming **and** outgoing (the first value is the preferred `map()` output) | Mainly an extra incoming entry point |
| Typical use | Localised page names while node names stay the same across language trees | Promotional shortcut: `/java-pro` → `/es/cursos/java-certification` |
| Regex | No | **No.** Not for dynamic routing |

Adobe guidance as the article states it: vanity paths **must be unique**,
**must not collide with existing page paths**, and are *explicit shortcuts,
not dynamic routing*. Compare Sling's `sling:vanityOrder`, which exists to
settle clashes between several resources that claim the same vanity path (see
the Sling doc). It is a tie-breaker, not a reason to allow clashes.

### Localised slugs, stable node names

```
/content/acme-learning/en/courses/java      → /en/courses/java
/content/acme-learning/es/courses/java      → /es/cursos/java       (courses: sling:alias=cursos)
/content/acme-learning/fr/courses/java      → /fr/formations/java   (courses: sling:alias=formations)
```

Keeping node names the same across language copies keeps MSM, translation and
code paths uniform, and the alias only changes the URL. An alias on the English
node (`courses`) would be redundant, because it equals the node name.

## Outgoing: always `map()`

```java
for (Page course : courses) {
    String courseUrl = resourceResolver.map(request, course.getPath());
    // use courseUrl in the rendered link
}
```

Never build a URL as `"/" + language + page.getPath()`. That hard-codes one
layer's rewrite into a component and bypasses aliases and mappings. Use the
`map(request, path)` overload so the mapping can take the host and scheme from
the request.

### Where outgoing mistakes show up

- **Canonical.** Aliases and vanity paths expose one page at several URLs,
  which splits SEO value. Emit `<link rel="canonical"
  href="https://www.example.com/es/cursos/java-certification"/>` and build it
  through `map()`.
- **Sitemap.** The Apache Sling Sitemap module produces absolute URLs through
  Sling mappings (`JCR resource → Sling mapping → external URL → entry`). If
  the sitemap has `/content/...` paths or raw node names instead of the
  aliases, outgoing mapping is broken, and it is the quickest place to spot it.

## Debugging on AEMaaCS

| Where | Tool |
|---|---|
| Local SDK | `/system/console/jcrresolver`: interactive **Resolve** and **Map** boxes. Test both directions. |
| AEMaaCS Author/Publish | `/system/console` is **not exposed**. The Developer Console (from Cloud Manager) shows OSGi configs, bundles, components and servlets **read-only, with no Resolve/Map tester**. |

So reproduce the mapping on the SDK, then check the cloud environment from
the outside, layer by layer:

```
CDN/Apache response (status, Location) → Dispatcher rewrite logs → AEM resolution
  → rendered HTML links → canonical → sitemap
```

Keep resolver OSGi configuration in code and deploy it through Cloud Manager.
Never change it in a console, which you can't reach on AEMaaCS anyway. See
[aemaacs-repository-inspection-by-tier.md](aemaacs-repository-inspection-by-tier.md)
for which tool is available on which tier.

## Picking a mechanism

| Requirement | Mechanism |
|---|---|
| Hide `/content` prefix | Apache rewrite (incoming) **plus** Sling mapping (outgoing). You need both. |
| Localise a segment, keep the JCR name | `sling:alias` |
| Promotional shortcut | Vanity path |
| Permanent URL move | HTTP 301 |
| Render links from JCR paths | `ResourceResolver.map()` |
| Absolute sitemap URLs | Sling mappings |
| Let Dispatcher serve vanity paths | `/vanity_urls` farm block |
| Debug incoming / outgoing | `jcrresolver` Resolve / Map (SDK only) |

## Common mistakes

1. Treating `sling:alias` and vanity paths as the same thing.
2. Putting every URL rule in Dispatcher.
3. Testing `resolve()` only, never `map()`.
4. Building link strings by hand instead of calling `map()`.
5. Using vanity paths as a regex router.
6. A URL design where cached paths and flush paths differ, so pages go stale
   with no error.

## References

- Experience League Community, *Understanding URL Resolution in AEM as a Cloud
  Service: Sling Mappings, Aliases, Vanity URLs and Dispatcher*:
  https://experienceleaguecommunities.adobe.com/adobe-experience-manager-sites-8/understanding-url-resolution-in-aem-as-a-cloud-service-sling-mappings-aliases-vanity-urls-and-dispatcher-252910
- [sling-resource-resolution-mapping.md](sling-resource-resolution-mapping.md)
- [aemaacs-repository-inspection-by-tier.md](aemaacs-repository-inspection-by-tier.md)
- [dispatcher-ignoreurlparams.md](dispatcher-ignoreurlparams.md) (the other place where cache keys and public URLs drift apart)
