# AEM Assets Brand Portal

A **cloud-hosted (SaaS) companion to AEM Assets** for securely distributing approved brand assets to people **outside** your AEM environment — agencies, partners, distributors, resellers, internal field/sales teams — without giving them access to your AEM Author instance. Source: [Adobe — Brand Portal docs](https://experienceleague.adobe.com/en/docs/experience-manager-brand-portal/using/introduction/introduction).

---

## The problem it solves

You need to share a curated set of approved assets with a **large, external audience** and (optionally) let them **contribute** new creatives back — but:

- You do **not** want external users touching AEM Author (unpublished content, workflows, admin functions).
- You do **not** want to onboard thousands of external users into your internal **Active Directory / LDAP** (cost + management burden).
- Different external groups (e.g. competing agencies) must be **isolated from each other** — each sees only what's shared with them.

Brand Portal is purpose-built for exactly this. Think of it as a curated, cloud-hosted "asset storefront + drop box" that sits in front of your DAM.

## What it is

- A **separate Adobe-hosted cloud tenant**, isolated from your Author/Publish instances. That isolation is precisely why it's safe for external users.
- **Tightly coupled to AEM Assets** — it's an extension of the Assets DAM, not a standalone product. You publish approved assets *out* from AEM Assets to the Brand Portal tenant.
- Identity is **Adobe IMS / Adobe ID**, managed in the **Adobe Admin Console** — no internal AD onboarding required. Scales to thousands of external users.

## Two directions of asset flow

| Direction | Feature | What it does |
|-----------|---------|--------------|
| **Outbound** (distribute) | **Publish to Brand Portal** | From AEM Assets you select assets/folders and publish them to the Brand Portal tenant. Only what you explicitly publish is visible. |
| **Inbound** (contribute) | **Asset Sourcing** | External contributors **upload** new creatives into designated **contribution folders**, which flow back into AEM Assets for review/approval. This is what makes Brand Portal a two-way contribution platform, not just a distribution channel. |

**Exam tip:** "Brand Portal **with Asset Sourcing**" is the standard answer when the scenario needs external partners to *contribute* (upload), not just download. Plain Brand Portal alone = distribution only.

## Key capabilities

- **Folder-level access control / user groups** — assign each agency/partner to a group that sees only its shared folders → groups **cannot see each other's work**.
- **Scales to thousands of external users** with no AD onboarding (Adobe IMS identity).
- Search, **Collections**, download presets/renditions, asset **expiry**, portal **branding/theming**.
- Complete **isolation from internal AEM** (Author/Publish never exposed).

## Where you get it / licensing

- It's an **Adobe-licensed add-on / entitlement**, **provisioned by Adobe** — you don't install it. Contact your **Adobe account team / sales rep** to have a Brand Portal tenant provisioned.
- **Bundled with some AEM Assets licenses** (notably AEM Assets as a Cloud Service and certain Assets editions include a Brand Portal entitlement), but inclusion depends on your specific contract → in practice, **check your Adobe entitlements / ask your account team**.
- Administered via the **Adobe Admin Console** (provision tenant, manage users/groups).

## When to use it vs alternatives (the exam traps)

| Approach | Why it's usually wrong for external contributors |
|----------|--------------------------------------------------|
| Local accounts on **AEM Author** | Exposes unpublished content, workflows, admin functions to external users; managing thousands of accounts + ACLs is complex and a security risk. |
| Local accounts on **AEM Publisher** (+ reverse replication) | Publisher serves published content to end users, not a contribution platform; weak access isolation at scale; reverse-replication complexity. |
| **Hot folder** file sync | No authentication, no access control, no per-partner isolation. |
| **Brand Portal + Asset Sourcing** | ✅ Purpose-built: external distribution **and** contribution, folder-level isolation, no AD onboarding, scales to thousands, isolated from internal AEM. |

**Rule of thumb:** *External contributor + many partners + don't expose internal AEM* → **Brand Portal (+ Asset Sourcing)**. Never put external users on Author.