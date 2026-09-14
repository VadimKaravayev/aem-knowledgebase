# Escaping `<` and `>` in Vault `.content.xml` Attribute Values

How to safely embed HTML-bearing strings (RTE bodies, default `_cq_template` content) inside vault XML attributes without either failing the parser or shipping literal `&lt;` / `&gt;` text into the running JCR.

The short version: **escape `<` to `&lt;`. Leave `>` as a literal `>`. Always escape `&` to `&amp;`.**

---

## The XML 1.0 spec position

Inside an attribute value, only three characters *must* be escaped:

| Char | Escape | Required because |
|---|---|---|
| `&` | `&amp;` | Always. Otherwise the parser tries to read it as the start of an entity reference. |
| `<` | `&lt;` | Always. `<` is forbidden in attribute values because it would terminate the start tag. |
| `"` or `'` | `&quot;` / `&apos;` | Only the quote character that delimits the attribute itself. The other is fine as a literal. |

`>` is **not** on that list. The W3C spec (§2.4 *Character Data and Markup*) allows a literal `>` inside attribute values; the requirement to escape it only applies in element content (CDATA), and only when it would form the sequence `]]>`. Most XML authoring tools escape `>` anyway by convention, but it isn't required.

---

## The AEM gotcha — escape `<` only, not `>`

When a vault `.content.xml` seeds a property that AEM treats as **rich-text HTML** (e.g. a Core Text `text` value, or any property later rendered through `${value @ context='html'}` in HTL), escape only `<`:

```xml
<!-- ✓ works — renders as two paragraphs in the RTE -->
role="&lt;p>VP Consulting Solutions&lt;/p>&lt;p>Data &amp; AI&lt;/p>"

<!-- ✘ XML parser rejects the file at install time -->
role="<p>VP Consulting Solutions</p><p>Data &amp; AI</p>"

<!-- ✘ installs fine, but the RTE renders the literal text "&lt;p&gt;...&lt;/p&gt;" -->
role="&lt;p&gt;VP Consulting Solutions&lt;/p&gt;&lt;p&gt;Data &amp; AI&lt;/p&gt;"
```

### Why the third form fails

It looks canonical — `&lt;p&gt;...` is exactly what most XML serializers produce — but it ships literal entity text into the JCR property as `<p>...</p>` *that's been double-escaped*. The path the value travels:

1. **Vault XML parser** decodes `&lt;p&gt;` → `<p>`. So far so good.
2. **JCR persists** the string `<p>...</p>`. Fine.
3. **RTE clientlib** reads the stored string. If the stored string contains `&gt;` literally (because vault left it un-decoded as a side effect of how the property gets normalized), the RTE displays the entity reference instead of treating it as a tag closer.

Empirically: leaving `>` unescaped in the XML source produces a JCR value with a literal `>`, which the RTE renders correctly as the closing of `<p>`. Escaping it to `&gt;` produces a JCR value where `>` arrives in some pipeline as `&gt;`, and the RTE renders that as visible text.

This was confirmed on a real KFORCE component (`profilecard` default `role`) in May 2026. The canonical form failed; the asymmetric form worked.

---

## What `>` *not* being escaped means in practice

`>` inside an attribute value is legal XML, so every conforming parser accepts it. IDEs that auto-format XML may try to "fix" your file by re-escaping `>` → `&gt;` — turn off that rule or be ready to re-revert. In this repo, configure your IDE's XML formatter to preserve attribute values verbatim.

---

## What else gets escaped

The rule of "escape `&` and `<`, leave `>` alone" applies to **everything** inside an attribute value: RTE/HTML defaults, multi-line text fields, anything containing literal characters that are XML-significant.

For attribute-value containing the attribute delimiter, you have two options:

- Switch the outer delimiter: `attr='value with "quotes"'` or `attr="value with 'apostrophes'"`. Vault generates double-quoted attributes by default; if you need a literal `"`, use `&quot;`.

---

## Related rules in this repo

- `aem-vault-multivalue-properties.md` (project rule) — covers the `String[]` vs `String` silent failure when you forget the `[…]` brackets. Different problem, same family of "vault XML attribute gotchas."
- `cq-template-default-content.md` (project rule) — gives the general structure of `_cq_template/.content.xml`. Its "Rich-text values" section needs the `<`-only-escape note added; until it is, this knowledgebase doc is the authoritative source.

---

## Reference

- W3C XML 1.0 §2.4: <https://www.w3.org/TR/REC-xml/#syntax>
- Apache Jackrabbit FileVault DocView: <https://jackrabbit.apache.org/filevault/docview.html>