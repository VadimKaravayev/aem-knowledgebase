# Content distribution on AEMaaCS — the journal, and why queues "block"

AEM as a Cloud Service does **not** use the 6.5 replication framework. Publication is
**Sling Content Distribution in journal mode** (`org.apache.sling.distribution.journal`):
author appends to an ordered, append-only log run as an Adobe pipeline service *outside*
the AEM runtime, and each publish pod is an independent subscriber that imports at its own
pace. Author never knows how many publishers exist — which is the whole point, because pods
come and go with autoscaling and on every deployment, and a new pod just replays from its
offset to catch up.

```
Author ──FileVault serialize (binary-less)──> content package
   │
   └──append──> aemdistribution_package (Kafka, 1 partition → strict order)
                         │
        ┌────────────────┼────────────────┐
     publish-1        publish-2        publish-N     each an independent subscriber
   (own offset)     (own offset)     (own offset)
        │
   read msg → import as sling-distribution-importer → write offset atomically with the data
        │
        └──> aemdistribution_status ──> Author's Distribution UI

large binaries: NOT in the package — referenced in the shared Oak blob store
```

## The five topics

| Topic | Carries |
|---|---|
| `aemdistribution_package` | the content packages themselves (protobuf; ~<10 KB typical, 800 KB max) |
| `aemdistribution_status` | per-subscriber import success/failure, feeding Author's Distribution UI |
| `aemdistribution_discovery` | subscriber heartbeats (~10 s) announcing identity + last processed offset |
| `aemdistribution_command` | clear/reset commands aimed at one subscriber |
| `aemdistribution_event` | JSON notification that a package was distributed |

`package` and `status` **support 1 partition only** — that is the mechanism that guarantees
ordering, and therefore the mechanism that causes blocking (below). `discovery` and `command`
partition by a hash of the subscriber agent name.

## Where the bytes actually go — two levels of indirection, don't conflate them

1. **Asset binaries** never travel in the package. FileVault serializes **in binary-less
   mode**, so the package holds *references* into the **shared Oak blob store** that author
   and publish both see. This is the same trick as
   [shared-datastore-binaryless-replication.md](shared-datastore-binaryless-replication.md),
   except there it is a 6.5 migration pattern you configure, and here it is the built-in
   architecture — the capacity mismatch between a Kafka-ish message bus and a 4 GB video is
   resolved by never putting the video on the bus.
2. **The package itself** normally travels *inside* the journal message. Only an
   **oversized** package is sent by reference: author parks it at
   `/var/sling/distribution/journal/packages` as a binary property backed by the shared blob
   store, and a cleanup job every 12 h drops packages below the current minimum offset.

The common mental model "author uploads packages to a shared store and publish downloads
them" is right only for the oversized case, and right about *binaries* always.

## Offsets and exactly-once

Consumer offsets are **not** managed by the log — *"consumer offsets for the Subscriber
agents are not managed by the persisted log, but by the agent code."* Each subscriber stores
its processed offset in its own repository under `/var/sling/distribution/journal/stores`,
written **in the same atomic commit as the imported content**, which is what makes import
exactly-once across pod restarts and replacements.

## Why a queue blocks — and why the symptom lies about its location

Strict ordering + per-subscriber offset = **head-of-line blocking**. A subscriber that cannot
import message *N* cannot advance past it. The queue stalls at that package and everything
activated afterwards piles up behind it.

The trap: **the blocked queue is displayed on Author, but the fault is on Publish.** Author
packaged and enqueued perfectly — that is *why* there is something in the queue to be stuck.
If the author side were at fault (e.g. the author-side agent could not read the content) the
failure would surface at **export** time and the queue would be **empty**, not blocked.

> Blocked queue with items in it → look at the importer on Publish.
> Nothing ever queued → look at the exporter on Author.

Unlike classic SCD, **the journal error queue does not allow retrying a failed item** — you
remove it. In a cloud deployment an instance repeatedly hitting a failed item is simply
replaced.

## Unblocking (Adobe's own runbook)

AEM Start → **Tools → Deployment → Distribution UI** → click the bolded row in the Queues
table → identify and record the path of the blocking item → remove it from the queue → fix
the content → republish. Needs admin permissions. Removing the item is what actually
unblocks; the remediation of the content is separate.

## The service users (a real naming trap)

Two sides, two principals — exam answers and log lines do not use the same name:

- **Publish, importing:** `sling-distribution-importer`, via the service user mapping
  `org.apache.sling.distribution.journal:importer`. Default grants cover `/content`, `/conf`,
  `/etc`. **Extend it with repoinit** (`ui.config`, PID
  `org.apache.sling.jcr.repoinit.RepositoryInitializer`) when distributing to a path it
  cannot write — see [repoinit-acls-on-apps-and-libs.md](repoinit-acls-on-apps-and-libs.md).
- **Author, distributing:** appears in logs as `userId=replication-service`.

Consequence worth knowing outside the exam: on Publish, the **only** session available during
import is the importer's, so `jcr:createdBy` / `jcr:lastModifiedBy` on published nodes read as
the distribution service user, not the author who made the change. There is no way to
impersonate the original author across that boundary — don't build features that depend on it.

## Exam signal vs. production reality (read both ways)

The certification framing is: *pipeline fails in Functional Testing → timeout replicating
`/content/...` → Distribution page shows queues blocked* ⇒ **the importer on the Publish
service lacks permissions on the target path.** Distractors name the right symptom location
("the dev environment", "the Author service") rather than the right tier.

In production, that inference is **much weaker**, and Adobe's own KBs are the counter-examples:

- `javax.jcr.AccessDeniedException` in a blocked queue is frequently **not** an ACL problem —
  Adobe: *"Despite the error being `javax.jcr.AccessDeniedException`, there is possibly no
  relation with the ACL / permissions."* One documented cause is a node carrying
  `jcr:lockIsDeep="true"` **without** `jcr:lockOwner`; the property *"should always be set
  conjointly with the `jcr:lockOwner`"*. Fix is a corrected content package, not a grant.
- `javax.jcr.NamespaceException: Unknown namespace prefix` blocks queues periodically
  (observed on DAM asset metadata nodes).
- **Content packages in the repository can block the queue by themselves** — both
  `accesscontroltool-content-package` and acs-aem-commons' `ui.content` shipped fixes
  specifically for blocking the AEMaaCS distribution queue. Suspect your own
  `ui.content`/`/etc/packages` payload before suspecting ACLs.

So: answer "permissions on Publish" on the exam; on a real incident, pull the blocking item's
path out of the Distribution UI and look at the *node* first.

---

Verified Sept 2026 from: Apache `sling-org-apache-sling-distribution-journal`
`docs/documentation.md` (topics, partition counts, binary-less mode, offset storage, error-queue
semantics); Adobe KB **ka-21668** (AccessDeniedException / `jcr:lockIsDeep`) and **ka-23465**
(NamespaceException, Distribution UI runbook); Netcentric accesscontroltool issue #566 and
acs-aem-commons commit `58f4f4f`. Service-user naming cross-read from community write-ups —
the `org.apache.sling.distribution.journal:importer` → `sling-distribution-importer` mapping is
consistent across sources, but I have **not** confirmed the author-side `replication-service`
mapping against an Adobe primary source. AEMaaCS only; none of this exists on 6.5/AMS, which
still uses replication agents.
