# repoinit ACLs on `/apps` and `/libs`: allowed, but ordered before your packages

## The myth to kill first

"`/apps` and `/libs` are immutable on AEMaaCS, therefore repoinit can't set
ACLs there — permissions for immutable paths have to ship as `rep:policy`
nodes inside `ui.apps`."

The premise is true and the conclusion does not follow. Adobe lists **"Add
ACLs"** among repoinit's supported operations with no path restriction, and
documents the `/apps`-and-`/libs` case explicitly — as an *ordering* caveat,
not a prohibition.

Immutability governs **runtime writes to content** under those trees. Access
control is applied by repoinit during repository initialization, which is not
the same thing and is not blocked.

## The real constraint: repoinit runs on a blank repository

> *"For ACLs defined for nodes underneath `/apps` or `/libs` the `repoinit`,
> execution starts on a blank repository. The packages are installed after
> `repoinit` so statements cannot rely on anything defined in the packages but
> must define the preconditions."*

Cloud Manager *"executes these statements, independently from the installation
of any content packages"* — and before them. So at the moment your ACL
statement runs, `/apps/myproject/components` does not exist yet. Your own
`ui.apps` package is what creates it, minutes later.

This is why the myth feels true in practice: people write the obvious
statement, it fails on a fresh environment because the path isn't there, and
they conclude the tree is off-limits.

### The failure is environment-dependent, which is the nasty part

On an environment that has already been deployed to, the paths exist from the
previous run, so the same statement succeeds. It breaks on a **fresh** or
rebuilt environment — new dev env, RDE reset, a restored stage. Classic
"works everywhere except the one that matters."

## Two ways to write it correctly

### Preferred: ACL high, scope with `rep:glob`

Adobe's own recommendation:

> *"For ACLs, the creation of deep structures might be cumbersome. Therefore,
> it is more reasonable to define an ACL on a higher level and constrain where
> it is supposed to act by way of a `rep:glob` restriction."*

```
set ACL for my-group
    allow jcr:read on /apps restriction(rep:glob,/msm/wcm/rolloutconfigs)
end
```

`/apps` always exists, so there is no precondition to satisfy and nothing to
create. This is the version to reach for by default.

### Alternative: create the path first

```
create path /apps/myproject/components(sling:Folder)
set ACL for my-group
    deny jcr:read on /apps/myproject/components
end
```

Works, but you are now declaring structure in two places — repoinit and the
package — and they can drift. Use it only when a `rep:glob` can't express the
scope.

## Other repoinit properties worth holding onto

- **Idempotent.** Scripts run at every startup; statements whose state already
  matches are skipped. Re-running is safe by design.
- **Atomic and explicit**, which is why Adobe prefers it to content packages
  for creating users, groups, service users, paths and ACLs.
- Full supported set: create/delete/disable **service users**, create/delete
  **groups**, create/delete **users**, **add ACLs**, **add paths**, **add CNDs**.
- Lives as `scripts` on an OSGi factory config for PID
  `org.apache.sling.jcr.repoinit.RepositoryInitializer`, e.g.
  `ui.config/src/main/content/jcr_root/apps/<project>/osgiconfig/config.author/org.apache.sling.jcr.repoinit.RepositoryInitializer~<name>.cfg.json`.
  Use the `config.author` / `config.publish` runmode folders when the
  permission should differ per tier — it usually should.

### `.config` vs `.cfg.json`, and `scripts` vs `references`

Two file formats are in circulation and both work; Adobe's *AEM Project Content
Package Structure* page prescribes the older `.config` format for repoinit
specifically:

```
ui.config/.../apps/<project>/osgiconfig/config.author/org.apache.sling.jcr.repoinit.RepositoryInitializer-author.config
```

The reason is legibility of multi-line scripts. `.config` takes the script as a
literal multi-line block inside `scripts=["..."]`; `.cfg.json` is JSON, so the
same script becomes a string array whose line breaks have to be escaped or
split element-by-element. The `.cfg.json` form shipped in the Translated
connector (below) installs fine — this is a readability preference, not a
compatibility rule.

The part that is **not** a preference: put the script inline in the `scripts`
property, not behind `references`. `references` points at a repository path,
which on a fresh environment is subject to the same "nothing exists yet"
ordering problem described above — the script itself has to be present in the
config.

## When package-carried ACEs still make sense

Shipping access control entries inside the content package remains valid, and
has the opposite ordering property: the ACE lands *with* the nodes it protects,
so preconditions are satisfied by construction. The tradeoff is that package
ACE behavior depends on the package's access-control handling mode and is
easier to get silently wrong on reinstall. Default to repoinit; reach for
package ACEs when an ACL is genuinely inseparable from the structure it
guards.

## Practical note on hiding a subtree from a group

Read ACLs really do control visibility in the tools people would use to snoop:
the AEMaaCS Repository Browser shows only *"resources and properties your user
has access to"*, and Granite renders navigation from the resource tree with the
requesting user's resolver — so denying `jcr:read` on a nav node makes the tile
disappear with no render-condition code. What ACLs do **not** cover: Package
Manager, the Developer Console bundle and OSGi component lists, and anyone
holding admin rights. See
[aemaacs-repository-inspection-by-tier.md](aemaacs-repository-inspection-by-tier.md)
for which of those exist on which tier.

## Confirmed in production code

The Translated AEM connector's own `RepositoryInitializer-service-users.config`
does both halves of this and ships to AEMaaCS:

```
# create structure required for ACLs
create path /var/translated(sling:OrderedFolder)
create path /conf/translated(sling:Folder)

create service user translated-service with path /home/users/system/translated-service

set ACL for translated-service
    allow jcr:read on /apps
    ...
end
```

Note `allow jcr:read on /apps` — a repoinit ACL on the immutable tree, working
in a shipped connector. And the comment on the first line is the precondition
rule stated in the file itself: the `create path` statements exist *because*
the ACL below them would otherwise run against nodes no package has created
yet.

Adobe docs read September 2026 (AEMaaCS deploying/overview); in-repo
confirmation from the Translated connector, same date.
