# Shared Datastore & Binary-less Replication

The standard AEM-in-public-cloud pattern for handling large DAM assets: point Author and Publish at the same blob store so replication doesn't have to move binaries.

---

## The problem it solves

Normally, Author and Publish each have their **own** datastore. Replicating an asset (upload, update, activation) means shipping the actual binary — image/video/PDF bytes — from Author's datastore into Publish's. With many business users frequently uploading/updating large DAM assets, this makes replication payload scale with **binary size**, which gets slow and expensive as volume grows — exactly the pattern in question 4 (~10 users, frequent asset churn, migrating to cloud for performance).

## How it works

1. **Shared datastore** — both Author and Publish instances are configured to use the **same** blob store (e.g. AWS S3, Azure Blob Storage) instead of separate local/TarMK datastores. A binary is written **once**, at upload time, into the shared store.
2. **Binary-less replication** — when a replication package is built, binaries are swapped for their **hash-code reference** instead of the raw bytes. The receiving instance (Publish) resolves that hash reference against the shared store it already has access to — since it's the *same* store, the binary is already there. Only **metadata and node references** actually travel over the replication wire.
3. Net effect: replication payload collapses from "megabytes/gigabytes per asset" to "a small metadata diff," independent of how large the underlying binary is.

## Configuration notes

- Requires a `secret` parameter (shared secret) configured identically on **every** instance connected to the shared datastore — this is what lets an instance trust hash references it didn't create itself.
- Cloud blob stores (S3, Azure Blob) are the natural fit — durable, reachable by multiple compute tiers, low failure probability vs. disk-based stores.
- On **AEMaaCS** this is the managed default — the platform runs the shared datastore for you; you don't hand-configure `S3DataStore` connector files like on 6.x AMS/on-prem.
- Migrating an existing on-prem instance to a shared S3 datastore is a **reconfigure + backfill** operation: stop AEM, back up the existing FileDataStore config, remove it, drop in the S3 connector + config, copy the local binaries up to S3, then restart against the new config.

## Operational tradeoff

Shared binaries mean **no storage isolation between tiers** — a binary referenced by both Author and Publish can't be garbage-collected until *neither* instance references it anymore. This makes **Data Store Garbage Collection** more complex to reason about: GC has to check references across every connected instance sharing the store before deleting a binary, not just the local repository.

## Exam signal

Scenario has: cloud migration + "improve performance" + a DAM with **many assets** + **frequent uploads/updates** by editors. The architectural answer is "shared datastore + binary-less replication," not scaling compute (MongoDB/horizontal scaling — solves a *traffic* problem, not a *binary replication* problem) or generic maintenance tasks (revision cleanup — housekeeping, not an architecture change). See [[dispatcher-ignoreurlparams]] for a similarly-shaped trap (config fix vs. bigger architectural rework) in a different subsystem.

## References
- [How to verify binary-less replication is working (Adobe KB, KA-16760)](https://experienceleague.adobe.com/docs/experience-cloud-kcs/kbarticles/KA-16760.html?lang=en)
- [Steps for migration of an existing AEM datastore to AWS S3 (Adobe KB, KA-16057)](https://experienceleague.adobe.com/docs/experience-cloud-kcs/kbarticles/KA-16057.html?lang=en)
- [Assets sizing guide — AEM 6.5 (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/assets/administer/assets-sizing-guide)
- Question 4 in this repo (`questions/question-4.md`) — the cloud-migration scenario this note was written from.
