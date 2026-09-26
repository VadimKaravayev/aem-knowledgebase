# Sling resource resolution: `/etc/map`, aliases, vanity paths, and the reverse-mapping trap

Two opposite operations share one configuration tree, and most mapping bugs are
really "I configured one direction and assumed the other came free."

- **Inbound — `ResourceResolver.resolve(…)`**: request URL → resource.
- **Outbound — `ResourceResolver.map(…)`**: resource path → URL. This is what
  renders every link AEM emits.

**Sourcing:** Apache Sling, *Mappings for Resource Resolution*, read September
2026. Not reproduced against a running instance. Inference and AEM-specific
additions are flagged where they appear.

Related: [ssl-termination-sling-mapping-404.md](ssl-termination-sling-mapping-404.md)
(the canonical scheme/port failure),
[osgi-config-runmode-resolution.md](osgi-config-runmode-resolution.md) (where
the resolver's OSGi config lives).
[url-resolution-layers-aemaacs.md](url-resolution-layers-aemaacs.md) (which
layer — Apache, Dispatcher, Sling — owns which URL rule on AEMaaCS; cache
invalidation and debugging without the console).

---

## The reverse-mapping trap

> *"Using regular expressions with wildcards in the Root Level Mappings
> prevent the respective entries from being used for reverse mappings."*

A wildcard root-level entry resolves inbound requests correctly and is
**silently unusable for `map()`**. Sling cannot run a wildcard backwards —
there is no single URL to reconstruct from a pattern.

The symptom is not a 404. Requests work; **links are wrong**. Every URL the
site renders comes out as the raw internal path (`/content/site/en/page.html`)
instead of the public one, because `map()` found no reversible entry and fell
through. People debug the templates.

If links must be rewritten, the entry has to be literal enough to invert. Test
both directions before believing a mapping works — see the console plugin at
the bottom.

---

## The tree, and why the node is called `www.example.com.443`

```
/etc/map
  +-- http | https
        +-- {host}.{port}
              +-- {mapping entry}
```

The inbound match is performed against a **constructed virtual path**:

```
{scheme}/{host}.{port}/{uri_path}
```

That construction is the whole explanation for the node naming. An HTTPS
request on 443 is matched as `https/www.example.com.443/…`, so it consults
`/etc/map/https/www.example.com.443` and nothing else — which is why an
instance that believes it is serving plain HTTP never looks at the `https`
subtree no matter how correct that subtree is. That failure is the subject of
[ssl-termination-sling-mapping-404.md](ssl-termination-sling-mapping-404.md).

### Entry properties

| Property | Meaning |
|---|---|
| `sling:match` | Regex to match against. Needed whenever the pattern contains characters a JCR **node name** cannot hold — that is what it exists for |
| `sling:internalRedirect` | Internal rewrite. **Multi-value: paths are tried in sequence** until one resolves |
| `sling:redirect` | External redirect — the client is sent back out and requests again |
| `sling:status` | Status for the redirect. Default **302**; permitted 300, 301, 302, 303, 307, 308 |

### Match order

1. The **longest matching entry string** is used.
2. Regular expressions are evaluated **in order**.
3. **First match wins.**

Same shape as the CORS policy selection in
[cors-policy-and-dispatcher.md](cors-policy-and-dispatcher.md): a broad entry
placed early shadows everything narrower after it, and the symptom is a later
entry that appears to do nothing.

### Environment-specific values via interpolation

```
$['env':'MY_HOST';default='www.example.com']
```

Types are `env` (environment variables), `prop` (bundle context properties) and
`config`. **Placeholders work in `sling:match` properties, not in JCR node
names** — which is exactly the constraint you would expect given node names
can't hold arbitrary characters.

Worth knowing against the deployment rules elsewhere in this KB: a content
package cannot be scoped to one environment
([rolling-deployment-two-version-overlap.md](rolling-deployment-two-version-overlap.md)),
so an `/etc/map` entry that reads an env var is one of the few seams where a
single shipped artifact can differ per environment.

*(Not from this page, flagged as AEM convention and unverified here:* AEM
projects commonly keep a separate publish-side tree and point the resolver at
it with `resource.resolver.map.location`, conventionally `/etc/map.publish`.
`/etc/map` is mutable content, so it ships in `ui.content` — see
[all-package-embed-structure.md](all-package-embed-structure.md).*)*

---

## `sling:alias`

A per-segment rename applied while drilling the tree: if a path segment has no
matching direct child, Sling looks for a child carrying that value in
`sling:alias`.

- **Multi-value.** The **first** value is the preferred name used for outbound
  `map()`; the rest resolve inbound only.
- **Simple names only** — no paths. Rejected: anything containing `/`, `?` or
  `#`, the values `.` and `..`, and the empty string.

### The `jcr:content` behaviour that surprises people

An alias set on a **`jcr:content` node applies to its parent**. And setting an
alias on *both* the parent and its `jcr:content` does not override — **the two
lists concatenate**. So a page ends up answering to the union of both sets,
which is rarely what someone editing one of them intended.

### Aliases and access control

An alias placed **directly on a protected resource may not be visible to
restricted principals**, so resolution fails for exactly the users the
protection was aimed at. The documented workaround is to place the alias on a
**child** resource under the protected parent instead.

### Performance

Alias resolution tests permutations while drilling, so it is a genuine cost on
the resolution path, not a lookup. The knobs:

| Property | Default | Effect |
|---|---|---|
| `resource.resolver.optimize_alias_resolution` | `true` | Caches alias resolution — keep it on |
| `resource.resolver.allowed_alias_locations` | — | Restricts where the optimization applies |
| `resource.resolver.alias_cache_in_background` | `true` | Builds the cache off the request path |

---

## Vanity paths

| Property | Meaning |
|---|---|
| `sling:vanityPath` | An alternative path that resolves to this resource |
| `sling:redirect` | Boolean — issue an HTTP redirect rather than resolving internally |
| `sling:redirectStatus` | Status for that redirect, default **302** |
| `sling:vanityOrder` | Precedence when **several resources claim the same vanity path** |

OSGi side:

| Property | Default | Effect |
|---|---|---|
| `resource.resolver.enable_vanitypath` | `true` | Master switch |
| `resource.resolver.vanitypath_allowlist` | — | Path prefixes where vanity paths are honoured |
| `resource.resolver.vanitypath_denylist` | — | Path prefixes where they are not |
| `resource.resolver.vanitypath_maxEntries` | `-1` (unlimited) | Cap on entries held |
| `resource.resolver.vanitypath_cache_in_background` | `true` | Build the map off the request path |
| `resource.resolver.default_vanity_redirect_status` | `302` | Default for `sling:redirect` |
| `resource.resolver.vanity_precedence` | `false` | Whether vanity paths are consulted before ordinary resolution |

*(Inference, not stated on the page:)* the allowlist/denylist and
`maxEntries` pair exists because vanity paths are collected by scanning the
repository for the property — an unbounded default over a large content tree is
a startup-cost and memory question, and confining the scan is the lever. Treat
that as the reasoning to verify before quoting it.

### The authentication hole — read this one twice

> *"One scenario where authentication requirements will not be registered
> properly is when the child of a protected resource has an external vanity
> path (or resource mapping) that is not a descendant of an existing
> authentication requirement."*

Stated plainly: a vanity path (or `/etc/map` entry) on a child of a protected
resource can **route around the authentication requirement** that protects it,
because the requirement is registered against the real path and the vanity
entry does not descend from it.

The documented mitigations are to use **external redirects** — so the client
re-requests the real, protected path and the requirement applies normally — or
to **register the authentication requirement manually** for the vanity path.

Worth auditing wherever vanity paths were handed to authors as a
self-service feature on a tree that also has protected branches. Nothing warns
you; the path simply works for someone who should have been challenged.

---

## Namespace mangling

Colons are not URL-friendly, so Sling rewrites them in both directions:

```
outbound  /content/jcr:content/jcr:data.png  →  /content/_jcr_content/_jcr_data.png
inbound   /content/_jcr_content/_jcr_data.png →  /content/jcr:content/jcr:data.png
```

**Only registered namespace prefixes are unmangled inbound.** A path segment
that merely looks like `_foo_bar` is left alone unless `foo` is a registered
prefix — so hand-built URLs containing underscores are not silently rewritten,
but hand-built URLs containing a *raw colon* are not what Sling emits either.
Let `map()` produce these rather than concatenating them yourself.

---

## Testing both directions

`/system/console/jcrresolver` (Felix console plugin) inspects the mapping and
resolver map entries, and — the useful part — evaluates **both**
`ResourceResolver.resolve(…)` and `ResourceResolver.map(…)` against a URL or
path you supply.

Given the reverse-mapping trap at the top, the habit worth forming is to check
a new mapping in both boxes before shipping it. A mapping that resolves is only
half-verified.

---

## References

- [Mappings for Resource Resolution (Apache Sling)](https://sling.apache.org/documentation/the-sling-engine/mappings-for-resource-resolution.html)
- [ssl-termination-sling-mapping-404.md](ssl-termination-sling-mapping-404.md)
- [cors-policy-and-dispatcher.md](cors-policy-and-dispatcher.md)
- [all-package-embed-structure.md](all-package-embed-structure.md)
