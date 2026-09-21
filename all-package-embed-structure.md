# The `all` container package: embed targets, structure packages, dependency traps

Everything here is about the *assembly* layer of an AEM Maven project — how
`ui.apps`, `ui.content`, `ui.config` and any vendor artifact get wired into the
single `all` container that Cloud Manager actually deploys. The four-package
split itself is well known; the wiring rules are where projects break, because
most of them are naming conventions with no validator behind them.

**Sourcing:** Adobe, *AEM Project Content Package Structure* (AEMaaCS), read
September 2026. Rules restated here are the ones that are non-obvious or whose
failure mode is silent; not reproduced against a live pipeline unless noted.

Related: [rolling-deployment-two-version-overlap.md](rolling-deployment-two-version-overlap.md)
(what lands when), [repoinit-acls-on-apps-and-libs.md](repoinit-acls-on-apps-and-libs.md)
(the `ui.config` payload), [oak-indexing-aemaacs.md](oak-indexing-aemaacs.md)
(the one package that breaks the `/apps`-only rule).

---

## The embed target grammar

`all/pom.xml` places each subpackage with an `<embedded>` block whose `<target>`
is a **four-level path with no free choices**:

```
/apps/<app-name>-packages/<type>/install<.runmode>?
  1        2                3        4
```

```xml
<embedded>
    <groupId>${project.groupId}</groupId>
    <artifactId>my-app.ui.apps</artifactId>
    <type>zip</type>
    <target>/apps/my-app-packages/application/install</target>
</embedded>
```

| Level | Value | Notes |
|---|---|---|
| 1 | `/apps` | fixed |
| 2 | `<app-name>-packages` | the `-packages` suffix is **load-bearing**, see below |
| 3 | `application` \| `content` \| `container` | a **type** folder, not a name you pick |
| 4 | `install`, `install.author`, `install.publish` | required; scope of installation |

Level 3 is chosen by the embedded package's own `packageType`, so it is not
free-form: a `packageType: content` artifact goes under `content/`, a
`packageType: application` artifact under `application/`, and a nested
container (a vendor `all`) under `container/`.

The container's own filter must cover the whole embed root, or the embedded
zips are built into the artifact and then not installed:

```xml
<!-- all/src/main/content/jcr_root/META-INF/vault/definition/filter.xml -->
<filter root="/apps/my-app-packages"/>
```

### Why `-packages`, and what happens without it

The suffix exists so that deployment code scanning `/apps/<app-name>` for
things to install never sees the folder holding the subpackages. Name the embed
root `/apps/my-app/...` instead, and the installer can treat the subpackages as
installables of the application it is currently installing — **cyclic
installation**, which Adobe describes as destructive. Nothing validates the
name; it is a convention whose only protection is that everyone follows it. Do
not "tidy" it away.

### `install.author` / `install.publish` are the only run-mode suffixes

Level 4 accepts no other run modes. This is narrower than the AEMaaCS run-mode
set generally (`author`/`publish` × `rde`/`dev`/`stage`/`prod`) — you can scope
an *OSGi config* per environment with `config.publish.prod`, but you cannot
scope a *package embed* to an environment, only to a tier. See
[rolling-deployment-two-version-overlap.md](rolling-deployment-two-version-overlap.md)
for the environment-scoping gap and the manual Package Manager fallback.

---

## Keep Cloud Manager off the subprojects

Only the `all` container is a deployment target. Every other module must carry:

```xml
<properties>
    <cloudManagerTarget>none</cloudManagerTarget>
</properties>
```

Without it Cloud Manager may deploy a subpackage independently of the
container, which reintroduces exactly the ordering and completeness problems
the container exists to prevent.

---

## The repository structure package (`ui.apps.structure`)

Every package with `packageType: application` must declare one in its FileVault
plugin config:

```xml
<repositoryStructurePackages>
    <repositoryStructurePackage>
        <groupId>${project.groupId}</groupId>
        <artifactId>ui.apps.structure</artifactId>
        <version>${project.version}</version>
    </repositoryStructurePackage>
</repositoryStructurePackages>
```

**What it is for:** it declares the structural roots the code packages live
under, so the build can prove no code package installs *over* another. It is a
build-time correctness check on filter roots, not a runtime artifact.

**`packageType: content` packages do not need one** — this asymmetry catches
people who copy the block into `ui.content` and then wonder why the structure
package has to know about `/content`.

It is also why the `validRoots` override in
[oak-indexing-aemaacs.md](oak-indexing-aemaacs.md) (gotcha #4) doesn't break
`/apps`: replacing the validator's valid-roots list still leaves `/apps` valid,
because the structure package dependency covers it. And per gotcha #6 there,
`/oak:index` must **not** be added to the structure package — a filter root
there claims ownership of the entire subtree and would wipe the OOTB indexes.

---

## Package dependencies — and the bundle-only exception

The general rule is unsurprising: a mutable content package should declare a
dependency on the immutable code package that renders it, so ordering is
explicit.

```
all (no dependencies)
├── common.ui.apps        (none)
├── site-a.ui.apps        (depends on common.ui.apps)
├── site-a.ui.content     (depends on site-a.ui.apps)
├── site-b.ui.apps        (depends on common.ui.apps)
└── site-b.ui.content     (depends on site-b.ui.apps)
```

```xml
<!-- ui.content/pom.xml -->
<dependencies>
    <dependency>
        <groupId>${project.groupId}</groupId>
        <artifactId>my-app.ui.apps</artifactId>
        <version>${project.version}</version>
    </dependency>
</dependencies>
```

### The trap

**A code package that contains only OSGi bundles must NOT be declared as a
dependency.** Bundle-only packages are not registered with AEM Package Manager,
so the dependency can never be satisfied and the dependent package fails to
install.

This is worth holding onto because the symptom reads like a version problem —
an unsatisfied dependency naming an artifact that demonstrably built and
deployed. The cause is that Package Manager has no record of it, not that the
version is wrong. Depend on the `ui.apps` that carries the components, never on
the `core` that carries only the jar.

---

## `ui.config` is a code package

`ui.config` carries `packageType: application` even though it contains no
components or scripts, because OSGi configuration **is** code. It is rooted at
an organizational folder under `/apps`:

```
/apps/my-app/osgiconfig/config                        <- defaults
/apps/my-app/osgiconfig/config.author                 <- tier-scoped
/apps/my-app/osgiconfig/config.publish.prod           <- tier + environment
```

This is where repoinit lives, as `RepositoryInitializer` factory configs — see
[repoinit-acls-on-apps-and-libs.md](repoinit-acls-on-apps-and-libs.md) for the
ordering rules that govern what those scripts can assume exists.

Adobe's distinction for what belongs in repoinit versus runtime: groups that
are **integral to the application's function** (a group a workflow assigns to)
ship as repoinit; organizational groupings are managed at runtime by admins.

---

## Shipping or consuming a vendor package

A third-party AEM application ships its **own `all` container**, and the
customer's `all` embeds it whole — this is what the `container` type folder at
level 3 is for:

```xml
<embedded>
    <groupId>com.vendor.x</groupId>
    <artifactId>vendor.plug-in.all</artifactId>
    <type>zip</type>
    <target>/apps/vendor-packages/container/install</target>
</embedded>
```

```
all (embeds)
├── core / ui.apps / ui.config / ui.content   (the customer's own)
├── vendor-x.all                              -> /apps/vendor-packages/container/install
└── vendor-y.all                              -> /apps/vendor-packages/container/install
```

Note the embed root is `/apps/vendor-packages`, shared across vendors, not one
root per vendor.

**Consequence for anyone shipping a connector:** the artifact a customer
integrates is the `all` zip, and every package nested inside it must itself
satisfy the AEMaaCS coding guidelines — Cloud Manager converts the whole tree
into Sling Feature Model artifacts, and one non-compliant embedded package
fails the customer's deploy, not yours. The customer has no way to fix it from
their side.

---

## Hard restrictions worth memorising

- **A single content package cannot deploy to both `/apps` and a runtime-writable
  area.** The split is per package, not per project. A mixed package installs
  only its mutable half, silently — see
  [rolling-deployment-two-version-overlap.md](rolling-deployment-two-version-overlap.md).
- **Container packages cannot use FileVault install hooks.** Neither can
  immutable packages. If you need one, you need a different mechanism.
- **Nothing deploys to `/libs`.** Product code only; overlay under `/apps/cq`,
  `/apps/dam` instead.
- **Oak indexes ship in a code package** despite `/oak:index` being mutable,
  because Cloud Manager must finish reindexing *before* switching the code
  image. Packaging them as content would put them on the wrong side of that
  ordering.
- **Same code to every environment.** Environment differences belong in
  run-mode-scoped OSGi config, not in differently-built artifacts — otherwise
  stage validates something production never runs.
- `<accessControlHandling>merge</accessControlHandling>` for packages that carry
  `rep:policy` nodes; see
  [repoinit-acls-on-apps-and-libs.md](repoinit-acls-on-apps-and-libs.md) for
  when package-carried ACEs are the right call at all.

---

## References

- [AEM Project Content Package Structure (Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/developing/aem-project-content-package-structure)
- [rolling-deployment-two-version-overlap.md](rolling-deployment-two-version-overlap.md)
- [repoinit-acls-on-apps-and-libs.md](repoinit-acls-on-apps-and-libs.md)
- [oak-indexing-aemaacs.md](oak-indexing-aemaacs.md)
