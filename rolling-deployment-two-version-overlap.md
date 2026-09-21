# AEMaaCS rolling deployments: designing for the two-version overlap

An AEMaaCS deploy is a rolling one. New nodes start and pass health checks while the **old version is
still serving traffic**, then old nodes retire. There is no moment when exactly one version of your
code is live, so any change that assumes "my new code and my new repository state arrive together"
is wrong. Mutable content is applied only after the application version has switched over.

This is the constraint that shapes how a connector ships service users, ACLs and indexes.

**Sourcing:** the Adobe *Deploying to AEM as a Cloud Service / Overview* page (Sept 2026),
interpreted for a vendor package. Not reproduced against a live pipeline.

---

## What lands, and when

Mutable content does not install in one step. Three phases, in this order:

| Phase | What installs |
|---|---|
| **Pre-startup** | Oak index definitions |
| **During startup** | Service users, ACLs, node types (this is where repoinit runs) |
| **Post-switchover** | Everything else, via Jackrabbit Vault |

Repoinit therefore runs on a repository where **no package content exists yet**, which is why ACLs
must target paths that already exist. See
[repoinit-acls-on-apps-and-libs.md](repoinit-acls-on-apps-and-libs.md).

## The three rules that follow

### 1. Spread service user and ACL changes over two releases

While the deploy is in flight, **old code is running against the new permissions**. Narrowing a
grant, renaming a service user, or moving a subservice mapping in the same release that updates the
code that depends on it means the old nodes lose access mid-deploy and throw `LoginException` or
`AccessDeniedException` until they retire.

The two-release shape:

- **Release N**: add the new grant / new service user / new mapping. Both old and new code work.
- **Release N+1**: switch the code over, and only then remove what is now unused.

### 2. A rollback does not roll back content

Rolling back restores the **code**. Mutable content structures created by the failed release stay in
the repository. So the old code has to tolerate structures it did not create, or you fix forward with
a new release. There is no automatic cleanup and no way to undo a mutable change through the
pipeline.

Practical reading for a connector: never write a mutable structure the previous version would choke
on (a new property with a value the old enum cannot parse, a node type the old code does not expect
under a path it scans). Make the old version's readers tolerant *before* the release that writes.

### 3. Indexes set the length of the deploy

A new or modified index definition triggers **reindexing before traffic shifts**, so deploy duration
tracks index size, not code size. A large index change can turn a routine deploy into a long one.
During the overlap, old nodes keep using the original index while new nodes use the modified one, so
both definitions must be serviceable at once. See
[oak-indexing-aemaacs.md](oak-indexing-aemaacs.md) for the `-custom-N` naming convention that makes
this work, and note that ACS Commons Ensure Oak Index is **not** usable on AEMaaCS.

## Reference facts worth not rediscovering

- **Run modes are a closed set.** `author` and `publish` crossed with `rde`, `dev`, `stage`, `prod`,
  combined as `<service>.<environment>` (`author.dev`, `publish.prod`). No custom run modes, and **no
  way to scope a package to one environment**; environment-specific content means a manual Package
  Manager install.
- **`cp2fm` packages are frozen.** Anything Cloud Manager deployed appears in Package Manager with a
  `cp2fm` suffix and cannot be rebuilt, reinstalled or downloaded.
- **Manual one-off installs are mutable-only and time out after 10 minutes.** Retrying a timed-out
  install can introduce conflicts.
- **No FileVault install hooks in immutable packages.**
- **A mixed immutable/mutable package installs only its mutable half**, silently. Keep the split
  clean in the `all` container.
- **OSGi config belongs in source control**, not the web console, and **maintenance task
  configuration must too**, because Tools → Operations is unavailable on AEMaaCS.
- Cloud Manager converts content packages into **Sling Feature Model** artifacts; every embedded
  third-party package must itself satisfy the cloud coding guidelines or the deploy fails.

## References

- [Deploying to AEM as a Cloud Service, Overview (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/deploying/overview)
- [repoinit-acls-on-apps-and-libs.md](repoinit-acls-on-apps-and-libs.md)
- [oak-indexing-aemaacs.md](oak-indexing-aemaacs.md)
- [cloud-manager-git-repositories.md](cloud-manager-git-repositories.md)
