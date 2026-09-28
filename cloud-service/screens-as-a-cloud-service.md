# AEM Screens as a Cloud Service: two halves, and the allow-list trap between them

Screens as a Cloud Service is digital signage for public displays, delivered as
an **add-on** to a Sites program. It is split into two components that run in
different places, and that split is the source of both the exam question and
the real-world integration gotcha.

---

## The two components

```
  AEM (AEMaaCS or AMS)                      Adobe I/O Runtime
 ┌────────────────────────────┐  channels.json  ┌────────────────────────────┐
 │ Screens CONTENT Provider   │ ──────────────▶ │ Screens SERVICES Provider  │
 │  add-on inside AEM         │  (publish tier) │  signage management SaaS   │
 │  projects, channels,       │                 │  displays, players,        │
 │  locations, content        │                 │  schedules, orchestration  │
 └────────────────────────────┘                 └─────────────┬──────────────┘
                                                              ▼
                                                        players / displays
```

| | Content Provider | Services Provider |
|---|---|---|
| Runs on | AEM as a Cloud Service **or Adobe Managed Services** | Adobe I/O Runtime, outside the AEM JVM |
| Who uses it | content authors | authors, developers, admins |
| Manages | **projects, channels, locations**, the content itself | **displays and players**, registration, where and when content plays |
| Reached via | Cloud Manager program → Environments card link → `/screens.html` on author | its own Experience Cloud UI, org-scoped |

The Content Provider deliberately hides displays and player registration from
authors. The Services Provider deliberately knows nothing about authoring. The
Services Provider is "the orchestrator": it tells players where and when
content plays, at a high level, by reading published channels from AEM.

Note the AMS mention: the Content Provider half can still be classic AEM 6.5
on AMS while the management half is the cloud service. That is the migration
path for existing Screens customers.

## Wiring them together

Configured in the Services Provider, gear icon next to Project, Edit Settings:

- **Publish URL** and **Author URL** of the AEM environment
  (`https://publish-p12345-e12345.adobeaemcloud.com` form).
- **At least one channel must already be created and published** before you
  save this connection, otherwise the connect fails. Create it at
  `/screens.html` on the Content Provider first.
- Log into the **correct IMS organisation** before doing any of this. The
  Services Provider is org-scoped and switching orgs is a top-right menu, easy
  to miss.

## The gotcha: IP Allow Lists break the Services Provider

The Services Provider pulls `/screens/channels.json` from your **publish**
tier from Adobe I/O Runtime IP space. If you have applied a Cloud Manager
**IP Allow List** to publish, that request is blocked and players get no
schedule.

Adobe's documented workaround is **not** to whitelist I/O Runtime IPs. Instead:

1. Move your allowed IPs out of the Cloud Manager IP Allow List and into a CDN
   traffic-filter rule in the config repository, then **unapply** the Cloud
   Manager list.
2. Add a second rule that **allows requests carrying a secret header** on the
   channels path.
3. Enter the same header key and value in the Services Provider settings.

```yaml
version: "1"
metadata:
  envTypes: ["dev", "stage", "prod"]
data:
  trafficFilters:
    rules:
      - name: "block-request-from-not-allowed-ips"
        when:
          allOf:
            - reqProperty: clientIp
              notIn: ["101.41.112.0/24"]
            - reqProperty: tier
              equals: publish
        action: block
      - name: "allow-requests-with-header"
        when:
          allOf:
            - reqProperty: tier
              equals: publish
            - reqProperty: path
              equals: /screens/channels.json
            - reqHeader: x-screens-allowlist-key
              equals: ${CDN_HEADER_KEY}
        action:
          type: allow
```

Keep the header value out of Git: reference it as a Cloud Manager secret
environment variable, which is what `${CDN_HEADER_KEY}` is. Adobe's sample
has an indentation error in the first rule's second condition; the version
above is corrected.

Why this matters beyond Screens: it is the general pattern for **letting one
trusted external service through a publish allow list** on AEMaaCS. Cloud
Manager IP Allow Lists are all-or-nothing per service; CDN traffic filters can
carve out a path plus header exception. See [[url-resolution-layers-aemaacs]]
for where the CDN rule layer sits.

## Exam and review checklist

- "Which component manages players and displays" is the **Services Provider**
  on I/O Runtime. "Which one do authors use for channels" is the **Content
  Provider** in AEM.
- Screens is an add-on to a Sites program, listed alongside Commerce in
  Cloud Manager's solution options. See [[cloud-manager-program-types]].
- Connecting the Services Provider before publishing a channel fails.
- Players stopped receiving content right after security hardened publish
  with an IP Allow List: the fix is a CDN header rule, not a support ticket.
- Screens on AMS with the cloud Services Provider is a valid hybrid.

## References
- [Introduction to AEM Screens as a Cloud Service (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/screens-as-cloud-service/overview/introduction)
- [Using Screens Content Provider (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/screens-as-cloud-service/configure-screens-cloud/using-screens-content-provider)
- [Setting up Screens Services Provider (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/screens-as-cloud-service/configure-screens-cloud/navigating-to-screens-services-provider)
- [Traffic filter rules including WAF rules (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/security/traffic-filter-rules-including-waf)
