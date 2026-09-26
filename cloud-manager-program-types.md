# Cloud Manager programs: hierarchy, production vs sandbox, and how to split solutions

Cloud Manager's entity hierarchy, the two program types, the full list of
sandbox restrictions, and Adobe's guidance on when one program should hold
Sites and Assets together versus apart. Sibling of
[[aemaacs-architecture-overview]], which covers the environments inside a
program.

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
| Deletable | **no**, only editable | yes |
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

### Hibernation and deployments

A hibernated sandbox is not a blocked sandbox. A Cloud Manager pipeline can
deploy to it while it stays hibernated, and the new code is live when someone
de-hibernates it. The Developer Console and Cloud Manager can both de-hibernate,
but neither is a deployment tool. If a requirement says "keep it hibernated but
deploy the code", the answer is "run the pipeline", not "wake it up first".
Watch the 3-month clock on anything you intend to keep.

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
- A sandbox that "disappeared" was hibernated three months.
- Support declining a sandbox runtime issue is expected; only sandbox creation
  and management are in scope.
- "Assets are mostly raw Creative Cloud files with their own approval flow"
  means separate Sites and Assets programs, likely with Connected Assets.

## References
- [Programs and program types (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/programs/program-types)
- [Introduction to sandbox programs (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/programs/introduction-sandbox-programs)
- [Introduction to production programs (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/programs/introduction-production-programs)
- Question 14 in `devops/questions/question-14.md` of the aem-architect-lab repo, the hibernated-sandbox deployment scenario.
