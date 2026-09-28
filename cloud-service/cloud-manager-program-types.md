# Cloud Manager programs: hierarchy, production vs sandbox, and how to split solutions

Cloud Manager's entity hierarchy, the two program types, the full list of
sandbox restrictions, hibernation mechanics, what editing a program can
change (solutions, WAF, CMK), how programs are deleted, and Adobe's guidance
on when one program should hold Sites and Assets together versus apart.
Sibling of [[aemaacs-architecture-overview]], which covers the environments
inside a program, and [[cloud-manager-ams-program-setup-kpis]] for the AMS
6.x flavour of program setup.

---

## The hierarchy

```
Tenant (IMS org, one per customer)
 └─ Program (one or more, usually one per licensed solution)
     ├─ Environments   exactly ONE production, any number of non-production
     ├─ Repository     auto-provisioned Git, plus any you add
     └─ Tools          pipelines, logs, monitoring, environment management
```

Adobe's own example: a tenant with two programs, a Sites program for a magazine
and an Assets program for its media library. Each program has its own dev,
stage and production environments. The rule to remember is **one production
environment per program**. If a scenario needs two independently released
production sites, it needs two programs.

Every program gets a Git repository provisioned automatically. You clone it,
commit, and push like any other remote. The remote just happens to live inside
Cloud Manager, and pipelines build from it. Additional repositories, including
customer-owned GitHub, can be attached later.

## Two program types

| | Production program | Sandbox program |
|---|---|---|
| Purpose | live traffic | training, demos, enablement, POCs, documentation |
| Solutions | whatever your contract licenses, mapped by you | Sites, Assets and Edge Delivery Services added automatically |
| Environments | 1 prod + 1 stage, 1 or 2 dev, 1 or 2 RDE | **one** dev environment only |
| Setup | creation wizard | auto-created: archetype sample project on a Git branch, dev environment, non-production pipeline |
| Deletable | yes, but **two-phase**: Business Owner marks for deletion, takedown period, then permanent removal; can be unmarked meanwhile | yes, immediately (removes all environments and pipelines) |
| SLA and commitments | yes, including 99.99% | none |

### Sandbox restrictions, in full

Sandbox programs are the exam's favourite source of "why can't I..." questions.
The complete list from Adobe:

- **No live traffic**, and therefore not covered by AEMaaCS commitments.
- **No auto-scaling**, so **not suitable for performance or load testing**.
- **No custom domains, no IP allow lists.**
- **No additional publish regions.**
- **No 99.99% SLA.**
- **No advanced networking**: no VPN, non-standard ports, or dedicated egress
  IP.
- **No automatic AEM updates.** Updates can be applied manually, but only to an
  environment with a properly configured pipeline. Updating either prod or
  stage updates the other, because that pair must stay on the same release.
- **No technical support** for issues inside the sandbox. Problems creating or
  managing the sandbox program itself are still supported.
- **Hibernation after 8 hours of inactivity. Deletion after 3 continuous
  months hibernated.**

### Hibernation, in detail

Hibernation is **sandbox-only**; production program environments never
hibernate.

- **Trigger**: automatic after **8 hours** with no requests to the **author,
  preview or publish** services. Manual hibernation exists (Developer Console
  → Hibernate) but is never required. Entering hibernation takes minutes and
  **data is preserved**.
- **Who can wake it**: anyone with a product profile granting access to AEM as
  a Cloud Service can open the Developer Console and click **De-hibernate**;
  Adobe calls out that a plain **Developer** role is enough. Waking is
  therefore not a Business Owner or Deployment Manager action.
- **What a visitor sees**: a browser request to author, preview or publish of
  a hibernated environment gets a **landing page** explaining the state with a
  link to the Developer Console.
- **Deployments and updates still work while hibernated.** A pipeline deploys
  custom code, and a **manual AEM release update** can also be triggered from
  Cloud Manager; in both cases the environment stays asleep and the new
  code/release is live when it is next de-hibernated. If a requirement says
  "keep it hibernated but deploy", the answer is "run the pipeline", not "wake
  it first".
- **The 3-month clock**: after **three continuous months** of hibernation the
  sandbox **environments** are deleted. The **sandbox program itself, its
  Git repository and code are retained**, and the environments can be
  recreated. "The sandbox disappeared" therefore means the environment,
  not the code.

## Editing a program (AEMaaCS)

Program Overview → click the program name → **Edit program**. Requires the
**Business Owner** role (also needed for the License Dashboard and for any
program deletion). The dialog exposes the same tabs as creation, plus:

- **Solutions**: add Sites to an Assets program or Assets to a Sites program,
  remove one when both are present, or attach an unused solution entitlement
  (or spend it on a new program). Solution and add-on changes **take effect
  after the next deployment**, not on Update.
- **Flexible Publish Tier (Beta)**: choose whether new environments get a
  publish tier provisioned at all.
- **Security tab, WAF-DDOS Protection**: a checkbox that is only meaningful if
  WAF rules are **licensed**; licensed-but-unchecked means the feature is
  **off**. Checking it gives some automatic CVE protection, but full protection
  still requires deploying traffic filter **WAF rules through a Cloud Manager
  config pipeline**. Proof that it is on: CDN log entries carry a `rules`
  property containing a `waf=` attribute, present even before any rule is
  deployed. Can be toggled on and off at any time.
- **Security tab, Customer Managed Keys (CMK)**: enable for an existing
  program, then configure the key in **Experience Hub** (Azure Key Vault
  details) via the program card's Configure CMK link. **CMK cannot be
  disabled once activated.** Environment details show a CMK status badge only
  once the key is actually configured for that environment.

## Deleting programs

**Production program: two-phase.** Program card → Delete program → the dialog
lists connected resources (prod, stage, dev environments); type the program
name → **Mark for deletion**.

1. Cloud Manager validates eligibility. Locked resources (an environment
   mid-update) disable the button; a failed validation lands the program in a
   *Failed to mark for deletion* state.
2. On success: **the program's credit is returned**, **all environments are
   removed**, and the card shows *Marked for deletion* with an Alert badge that
   reveals the scheduled permanent-removal date.
3. Until that date a Business Owner can **Unmark for deletion**, which
   **requires available credits** (the returned credit must still be free).
4. After the takedown period the program is permanently removed, no restore.

**Sandbox program: single step.** Delete Program from the program name menu or
the card's ellipsis; it removes all environments and pipelines. Business Owner
or Deployment Manager can instead delete just the sandbox's environments and
keep the program.

## Production program creation: how to map solutions to programs

Your contract fixes how many solutions you have. You choose how to map them to
programs. Adobe's decision table:

| Licensed | Recommended split | Environments | Choose this when |
|---|---|---|---|
| 1 Sites | one Sites-only program | 1 prod + 1 stage, 1 dev, 1 RDE | only option |
| 1 Assets | one Assets-only program | 1 prod + 1 stage, 1 dev, 1 RDE | only option |
| 1 Sites + 1 Assets | **one combined** Sites & Assets program | 1 prod + 1 stage, **2 dev, 2 RDE** | most assets exist to serve the sites, are finished, and one team manages both |
| 1 Sites + 1 Assets | **two separate** programs | each: 1 prod + 1 stage, 1 dev, 1 RDE | many assets do not support the site: raw shoots, PSD/AI work in progress, a creative team with its own approval workflow and release cycle. Consider Connected Assets. |
| 1 Sites + 1 Sites | two Sites-only programs | each: 1 prod + 1 stage, 1 dev, 1 RDE | multi-tenant: separate brands with their own release schedules and dev teams |

The heuristic behind the table: **split when lifecycles diverge**. Separate
teams, separate release cadence, or assets that live their own life before a
finished rendition reaches the site all argue for separate programs. Combining
buys two dev and two RDE environments in exchange for one shared release train.

Production programs may enable **Customer Managed Keys** for data at rest,
depending on entitlement.

## Exam and review checklist

- Two sites with independent release schedules: two programs, because a program
  has only one production environment.
- Load testing or performance testing on a sandbox is invalid twice over: no
  auto-scaling and no production-sized stage. Use stage in a production program.
- Sandbox is missing something operational (custom domain, IP allow list, VPN,
  extra region, SLA): that is by design, not a misconfiguration.
- A sandbox whose environments "disappeared" was hibernated three months; the
  program, repo and code are still there, recreate the environment.
- Anyone with AEM product access can de-hibernate; it is not an admin action.
- "Delete the production program" is a Business Owner two-phase action with a
  credit refund and a takedown window; unmarking needs the credit still free.
- Adding Assets to a Sites program is done in Edit program but is invisible
  until the next deployment.
- WAF licensed but no `waf=` in CDN logs: the Security-tab checkbox is off or
  the WAF rules were never deployed via the config pipeline.
- CMK is a one-way door; enabling it is a compliance decision, not a toggle.
- Support declining a sandbox runtime issue is expected; only sandbox creation
  and management are in scope.
- "Assets are mostly raw Creative Cloud files with their own approval flow"
  means separate Sites and Assets programs, likely with Connected Assets.

## References
- [Programs and program types (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/programs/program-types)
- [Introduction to sandbox programs (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/programs/introduction-sandbox-programs)
- [Introduction to production programs (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/programs/introduction-production-programs)
- [Edit programs (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/programs/editing-programs)
- [Hibernate and de-hibernate sandbox environments (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/programs/hibernating-environments)
- Question 14 in `devops/questions/question-14.md` of the aem-architect-lab repo, the hibernated-sandbox deployment scenario.
