# HTL `@ i18n` Uses the Page Language, Not the User's Locale

Why server-side HTL translations silently render in English on custom Granite console pages while the OOTB console chrome (search button, nav, etc.) shows the user's language — and how to fix it.

The short version: **HTL's `@ i18n` resolves its locale from the containing `cq:Page`'s language before it ever looks at the request. A console page under `/apps` has no language, so it defaults to English for every user. Pass the locale explicitly: `${'Text' @ i18n, locale=request.locale}`.**

---

## Symptom

- A custom console page (e.g. `granite/ui/components/shell/page` at `/apps/<app>/content/...`, reached via a vanity path) contains HTL like `${'Create new project' @ i18n}`.
- A valid i18n dictionary with the translations is deployed and visibly registered — the keys appear in `http://localhost:4502/libs/cq/i18n/dict.fr.json`.
- The OOTB Granite chrome on the very same page renders in the user's language (French), and the page even renders `<html lang="fr">`.
- Yet every `@ i18n` string renders in English — even when the request carries `Accept-Language: fr`.

Everything *looks* wired correctly, and everything except the HTL strings agrees the locale is French.

---

## Root cause — the locale lookup order

When no `locale` option is given, AEM's HTL i18n resolves the locale in this order:

1. the explicit `locale` option, if present;
2. **the language of the containing `cq:Page`** — `jcr:language` on `jcr:content` (walking up), else derived from a locale-looking path segment (`/content/site/en/...`), else the default → **English**;
3. the request locale — only when the resource is *not* inside a `cq:Page`.

Custom console pages are `cq:Page` nodes under `/apps/<app>/content/` with no `jcr:language` and no locale segment in the path → step 2 always resolves to English → step 3 (the actual user locale) is **never consulted**. The user's UI language, `Accept-Language`, user preferences — all ignored.

### Why the OOTB chrome still shows French

Granite translates its own console strings through different paths, both of which use the request/user locale:

- **server-side**: Granite UI JSPs use `com.day.cq.i18n.I18n(slingRequest)` → request locale;
- **client-side**: `Granite.I18n` fetches `/libs/cq/i18n/dict.<locale>.json` using the `<html lang>` attribute, which the shell sets from the user's locale.

So the shell chrome and your HTL strings genuinely run on **two different locales in the same request**. That's the confusing part: the mismatch is not client-vs-server caching or a broken dictionary — it's page-language-vs-request-locale.

---

## The fix

Pass the request locale explicitly on every `@ i18n` in console components:

```html
<p>${'No translation projects yet' @ i18n, locale=request.locale}</p>
<a href="...">${'Create new project' @ i18n, locale=request.locale}</a>
```

On author, `request.locale` reflects the signed-in user's UI language — the same source the Granite shell uses for `<html lang>` — so server-rendered HTL strings and the client-translated chrome finally agree.

**Rejected alternative:** setting `jcr:language` on the console page. That hardcodes one language for all users; a console must follow the *user's* locale, not the page's.

Confirmed on the Crowdin connector dashboard console (`/apps/crowdin/content/dashboard`, local AEMaaCS SDK quickstart), July 2026.

---

## Packaging a JSON-file i18n dictionary in `ui.apps` (works fine — not the culprit)

For reference, the dictionary setup that was suspected first but turned out to be correct. A JSON-file dictionary needs the `mix:language` mixin and `jcr:language` on the **nt:file node itself**, which in FileVault means a `<name>.json.dir/.content.xml` beside the file:

```
apps/<app>/i18n/
├── .content.xml            # nt:folder
├── fr.json                 # { "Create new project": "Créer un nouveau projet", ... }
└── fr.json.dir/
    └── .content.xml
```

```xml
<!-- fr.json.dir/.content.xml -->
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:mix="http://www.jcp.org/jcr/mix/1.0"
          xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
          jcr:language="fr"
          jcr:mixinTypes="[mix:language]"
          jcr:primaryType="nt:file">
    <jcr:content jcr:primaryType="nt:resource"/>
</jcr:root>
```

Sanity check after deploy: `curl -u admin:admin http://localhost:4502/libs/cq/i18n/dict.fr.json | grep '<your key>'`. If the key is there, the dictionary is registered and loaded — stop debugging the dictionary.

---

## Debugging recipe — isolating a "translation doesn't apply" problem

The efficient order, each step eliminating one layer (this is the sequence that found the root cause):

1. **Dictionary registered?** `GET /libs/cq/i18n/dict.<locale>.json` and grep for the key. Present → dictionary and packaging are fine.
2. **Locale reaching the server?** Render the page with `curl -H 'Accept-Language: fr'`. Check `<html lang="...">` in the output — if the shell says `lang="fr"` but your strings are English, the locale reaches the page; the problem is inside i18n resolution.
3. **Deployed script current?** Read the script straight off the instance (`GET /apps/.../<file>.html/jcr:content/jcr:data`) and diff against the repo — IDE sync tools can leave stale versions.
4. **Compiled HTL class current?** The FS classloader root is shown at `/system/console/fsclassloader` (e.g. `crx-quickstart/launchpad/felix/bundle<N>/data/classes/org/apache/sling/scripting/sightly/apps/...__002e__html.java`). Grep the generated `.java` for your `renderContext.call("i18n", ...)` calls. (Known gotcha: scripts synced without a `jcr:lastModified` bump keep serving the stale compiled class.)
5. **Isolate with a throwaway component.** Sling-POST a scratch component and render the same expression in different contexts:

   ```bash
   # script that prints the three variables that matter
   cat > tmptest.html <<'EOF'
   locale=[${request.locale}]
   plain=[${'Create new project' @ i18n}]
   withlocale=[${'Create new project' @ i18n, locale=request.locale}]
   EOF
   curl -u admin:admin -F"jcr:primaryType=sling:Folder" http://localhost:4502/apps/<app>/tmptest
   curl -u admin:admin -F"tmptest.html=@tmptest.html"   http://localhost:4502/apps/<app>/tmptest

   # render OUTSIDE any cq:Page
   curl -u admin:admin -F"sling:resourceType=<app>/tmptest" http://localhost:4502/content/tmptest
   curl -u admin:admin -H 'Accept-Language: fr' http://localhost:4502/content/tmptest.html

   # render INSIDE a cq:Page
   curl -u admin:admin -F"jcr:primaryType=cq:Page" http://localhost:4502/content/tmptest3
   curl -u admin:admin -F"jcr:primaryType=nt:unstructured" http://localhost:4502/content/tmptest3/jcr:content
   curl -u admin:admin -F"sling:resourceType=<app>/tmptest" http://localhost:4502/content/tmptest3/jcr:content/par
   curl -u admin:admin -H 'Accept-Language: fr' http://localhost:4502/content/tmptest3/jcr:content/par.html
   ```

   The signature result that nails this gotcha:

   | Context | `plain` | `withlocale` |
   |---|---|---|
   | outside any `cq:Page` | Créer un nouveau projet | Créer un nouveau projet |
   | inside a `cq:Page` | **Create new project** | Créer un nouveau projet |

   Clean up with `curl -u admin:admin -F":operation=delete" <url>` on each test node.

---

## Related docs in this knowledgebase

- `shell-page-vs-foundation-page.md` — building the custom console pages where this gotcha bites.
- `translation-rules-xml.md` — content translation (sending content out); unrelated to UI-string i18n, don't confuse the two layers.