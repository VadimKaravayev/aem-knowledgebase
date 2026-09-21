# SSL termination at load balancer + Sling Mapping 404s

## The problem

Site goes live over HTTPS with SSL terminated at the load balancer (not at AEM). The LB sets `X-Forwarded-Proto: https` and forwards plain HTTP to AEM. Sling Mapping only exists under `/etc/map/http/www.domain.com.80` (carried over from lower environments tested without SSL). Result: **all requests 404** in production.

## Root cause (two parts, both required)

1. **AEM's Felix HTTP Service SSL Filter isn't configured to trust `X-Forwarded-Proto`.** Since the LB→AEM connection is plain HTTP, AEM's own `request.getScheme()` / `request.isSecure()` defaults to `http` — it never inspects the forwarded header unless told to.
2. **Sling Mapping only has an `http` entry, not `https`.** Even if AEM did know the request was HTTPS, mapping resolution looks under `/etc/map/https/www.domain.com.443` for a 443 HTTPS request — that node doesn't exist, only `/etc/map/http/...80` does.

   That lookup path isn't arbitrary: Sling matches against a **constructed virtual path** `{scheme}/{host}.{port}/{uri_path}`, so the scheme AEM believes it is serving picks the `/etc/map` subtree before any entry inside it is considered. See [sling-resource-resolution-mapping.md](sling-resource-resolution-mapping.md) for the construction rule, entry matching order, and the separate trap where wildcard entries resolve inbound but can't be reversed for link rewriting.

Both must be fixed together. Fixing only the Sling Mapping (creating `/etc/map/https/www.domain.com.443`) does **nothing** on its own if AEM never learns the request was HTTPS in the first place — it keeps resolving against the `http` map root regardless of what's under `https`.

Fix:
- Configure the **Apache Felix HTTP Service SSL Filter** to read `X-Forwarded-Proto` (NOT the Adobe Granite SSL Connector Factory — that's only for when AEM itself terminates SSL directly, which isn't the case when a LB terminates it).
- Create the Sling Mapping at `/etc/map/https/www.domain.com.443`.

## Plain-English analogy

A building has one internal hallway. A **guard** at the front door checks each visitor's badge (regular=HTTP or VIP=HTTPS), then writes a note on their back saying which badge they had — but in a coded symbol, not plain writing (= the `X-Forwarded-Proto` header), and lets them through. But **every visitor walks the same single hallway** inside — so by the time they reach the **receptionist**, they all look identical; she can't see badges or notes and defaults to assuming everyone is "regular."

A **translator** stands just past the door specifically to read the note on each visitor's back and call ahead: "treat this one as VIP." But the translator can only decode the note if they have the matching **codebook** — that codebook is the SSL Filter's config, which has to explicitly name which symbol/header (`X-Forwarded-Proto`) means "VIP." Without that codebook configured, the translator sees a note but can't read it, stays silent, and the receptionist never hears "VIP" — she just defaults to her regular list, even if her VIP list (`/etc/map/https/...443`) is sitting right there, ready and correct. She never opens it because nobody flagged the visitor as VIP in the first place.

Source: [question-45.md](../my-own-repos/aem-architect-lab/questions/question-45.md)
