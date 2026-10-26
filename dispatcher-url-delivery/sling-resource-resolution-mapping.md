# Sling resource resolution: `/etc/map`, aliases, vanity paths, and the reverse-mapping trap

Two opposite operations share one configuration tree, and most mapping bugs are
really "I configured one direction and assumed the other came free."

- **Inbound — `ResourceResolver.resolve(…)`**: request URL → resource.
- **Outbound — `ResourceResolver.map(…)`**: resource path → URL. This is what
  renders every link AEM emits.

**Sourcing:** Apache Sling, *Mappings for Resource Resolution*
(this doc is the knowledgebase's home for that page). Not reproduced
against a running instance. Inference and AEM-specific
additions are flagged where they appear.

Related: [ssl-termination-sling-mapping-404.md](ssl-termination-sling-mapping-404.md)
(the canonical scheme/port failure),
[osgi-config-runmode-resolution.md](../infrastructure-ops/osgi-config-runmode-resolution.md) (where
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
([rolling-deployment-two-version-overlap.md](../cloud-service/rolling-deployment-two-version-overlap.md)),
so an `/etc/map` entry that reads an env var is one of the few seams where a
single shipped artifact can differ per environment.

*(Not from this page, flagged as AEM convention and unverified here:* AEM
projects commonly keep a separate publish-side tree and point the resolver at
it with `resource.resolver.map.location`, conventionally `/etc/map.publish`.
`/etc/map` is mutable content, so it ships in `ui.content` — see
[all-package-embed-structure.md](../java-osgi-build/all-package-embed-structure.md).*)*

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

## The trap's mirror image: regex in `sling:internalRedirect` = reverse-only (SLING-2560)

The rule above says a wildcard in the *match* side kills reverse mapping. The inverse is also
true and is the sanctioned way to get a regex-based **outbound** rule: **if `sling:internalRedirect`
contains a regex, the entry is used only for `map()` and never for `resolve()`.** The node then
carries a `sling:match` whose `$1`/`$2` refer to the groups captured in `sling:internalRedirect`:

```
/etc/map/http/example.com.80
    sling:internalRedirect = "/content/([^/]+)/home/(.*)"
    sling:match            = "$1/index/$2"
```

Reverse table entry: pattern `/content/([^/]+)/home/(.*)` → replacement `https://example.com/$1/index/$2`.
So the two regex placements are complementary, not interchangeable: **regex in `sling:match` → inbound
only; regex in `sling:internalRedirect` → outbound only.** A site that needs both directions
pattern-driven needs two entries.

## Page leftovers worth one line each

- **What `map()` returns**: an absolute URL (scheme + host + port) only when a resource-based entry
  exists with no `sling:match`, or with a `sling:match` that contains no wildcard; otherwise an
  absolute path. The `map(request, path)` variants additionally return a URL only when the request's
  `Host` header differs from the mapped host:port — same-host links stay path-relative. Multiple
  outgoing matches resolve unpredictably, so **never define conflicting `sling:match` values under
  different schemes/hosts/ports**.
- **Only nodes carrying `sling:redirect` or `sling:internalRedirect` become table entries**; every
  other node under `/etc/map` is pure structure for the regex built from the hierarchy.
- **A replacement that is a URL is fed back through the table** until the result is a path or nothing
  matches — chained entries work, and a loop between two entries is possible.
- **After the table**: a relative result is combined with the resolver search path
  (`ResourceResolver.getSearchPath()`), first hit wins; an absolute one is used as-is. Then four cases:
  *star resource* (`/*` — `SlingPostServlet` creation target), *direct existence check*, *drill-down*
  (child by name, else child by `sling:alias`, else stop), *missing resource*.
- **`resource.resolver.mapping`** (OSGi) defines host-agnostic mappings with separate inbound and
  outbound rules; **only `/etc/map` entries can be per host**. Both sources compile into the same two
  internal tables.
- **Alias cache contradiction on the page**: `optimize_alias_resolution` is marked *deprecated,
  disable it* in the property table, yet the performance section says disabling it is *not
  recommended* (SLING-12025: speed and `jcr:content` behaviour differ). Read it as "the switch is
  going away because the cache becomes mandatory", and leave it on; expect slow startup/alias updates
  past ~10 000 aliases. `allowed_alias_locations` only limits **cache initialisation**, not
  resolution. Invalid aliases are logged at **WARN**.
- Vanity extras: `vanitypath_maxEntries_startup=true` applies the cap only while the cache is built;
  `vanitypath_bloomfilter_maxBytes=1024000` is documented as *do not touch*.
- **Node types** `sling:Mapping` (primary, extends `nt:hierarchyNode` so it sits under `nt:folder`),
  `sling:MappingSpec` and `sling:ResourceAlias` (mixins) exist for convenience — **the implementation
  reads only the properties, never the types**, so `sling:Folder` nodes with the right properties work.
- String interpolation (SLING-7768) placeholder grammar is `$['type':'name';default='value']`, values
  come from the **Apache Sling String Interpolation Provider** config for `config`, env vars for `env`,
  framework-then-system properties for `prop`; works anywhere in the `/etc/map` tree.

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
- [all-package-embed-structure.md](../java-osgi-build/all-package-embed-structure.md)
