# Where a run mode comes from: the five sources, and which half is mutable

Two separate questions people run together: *how* does an instance acquire its
run modes, and *which* of them can still be changed. The answers differ per run
mode, and the mutable half is the one the KB has so far left implicit.

**Sourcing:** Adobe, *Run Modes* (AEM **6.5**), read September 2026. The
precedence list is as documented; not reproduced against a running instance.
Interpretation is flagged where it appears.

Related: [production-ready-mode-crxde.md](production-ready-mode-crxde.md) (the
frozen half, and what it costs you),
[osgi-config-runmode-resolution.md](osgi-config-runmode-resolution.md) (how
`config.<runmode>` folders resolve once the modes are set).

---

## The five sources

| # | Mechanism | Form |
|---|---|---|
| 1 | JVM system property | `-Dsling.run.modes=publish,prod,us` |
| 2 | `sling.properties` | `sling.run.modes=author` in `crx-quickstart/conf/sling.properties` |
| 3 | Start option | `java -jar cq-56-p4545.jar -r dev` |
| 4 | Jar filename | `cq5-<run-mode>-p<port-number>.jar`, e.g. `cq5-publish-p4503.jar` |
| 5 | `web.xml` | `sling.run.modes` property, application-server deployments only |

**Documented precedence**, highest first:

1. System properties (`-D`)
2. `sling.properties`
3. `-r` option
4. Filename detection

### Reading the precedence list honestly

That ordering is most meaningful on the **first** start. Afterwards
`sling.properties` already contains what was resolved then, and the
installation run modes it holds are frozen regardless of what you pass on a
later start — so "`-D` beats `sling.properties`" does not mean you can flip an
installed author into a publish with a `-D` flag. It cannot. What a later `-D`
or `-r` can still do is add or drop the *custom* modes described below.

*(That framing is this doc's reading of how the precedence rule and the
freezing rule interact, not a quote — the page states each separately.)*

The AEMaaCS SDK quickstart uses mechanism 4 exclusively, with its own filename
grammar (`aem-author-p4502.jar`, `aem-publish_stage-p4503.jar`); see
[production-ready-mode-crxde.md](production-ready-mode-crxde.md).

---

## Frozen versus re-selected

**Installation run modes — fixed at first start, forever.** Two mutually
exclusive pairs; you get one from each:

- `author` | `publish`
- `samplecontent` | `nosamplecontent`

Changing the start command later does nothing. The only exit is deleting
`crx-quickstart/` and reinstalling — which is why `nosamplecontent` hardening
cannot be undone in place, the subject of
[production-ready-mode-crxde.md](production-ready-mode-crxde.md).

The single documented exception runs in the other direction and only during a
**TarMK `crx2oak` migration**, whose `--promote-runmode nosamplecontent` flag
lets an instance upgraded from a pre-6.3 source acquire a mode it never started
with. It rebuilds the repository to do it, so it is not a way to change a mode
in place, and it does not exist for 6.3+ or MongoMK sources, where no migration
runs. See [in-place-upgrade-6x-to-65.md](in-place-upgrade-6x-to-65.md).

**Custom run modes — re-selected at every startup.** Anything that is not one
of the four above is free: `development`, `test`, `golive`, `intranet`,
`dev`/`stage`/`prod`, region tags like `emea`. They layer onto the frozen
installation mode:

- `author` + `development`
- `publish` + `test` + `golive`
- `publish` + `intranet`

So an instance's mode set is one fixed tier, one fixed sample-content decision,
and a freely mutable tail. Design anything that has to be switchable —
endpoints, feature toggles, log levels — against the tail.

Adobe's own caveat on the frozen half is narrower than it first reads:
*"changes to the individual configuration properties will take effect upon
restart."* The **mode** is frozen; the **configuration** selected by that mode
is not.

### AEMaaCS has no mutable tail

On AEMaaCS the run-mode set is closed — `author`/`publish` crossed with
`dev`/`stage`/`prod`/`rde`, as `<service>.<environment>` — and custom run modes
do not exist. The 6.5 habit of inventing a run mode per concern does not port.
See [rolling-deployment-two-version-overlap.md](rolling-deployment-two-version-overlap.md).

---

## `install.<runmode>` is not `config.<runmode>`

Two parallel folder conventions under `/apps/<project>`, resolved by the same
run-mode matching but carrying different payloads:

| Folder | Node type | Holds |
|---|---|---|
| `config.<runmode>` | `sling:Folder` | OSGi configurations |
| `install.<runmode>` | `nt:folder` | **bundles and packages** |

`install.author` and `install.publish` are the common cases, but on 6.5
`install.<anyCustomRunmode>` works too — you can ship a bundle only to
instances started with `dev`, or only to `intranet`.

**The cloud divergence worth holding onto:** on AEMaaCS, the `install` folders
that appear in an `all` container's embed targets accept **only** `install`,
`install.author` and `install.publish` — no custom suffix, because there are no
custom run modes. That restriction belongs to the cloud embed grammar, not to
AEM generally; see
[all-package-embed-structure.md](all-package-embed-structure.md). Code carried
across from a 6.5 project that relied on `install.dev` has nowhere to land.

For how competing `config.` folders resolve against each other — including the
rule that the most specific match replaces the others for the entire PID rather
than merging with them — see
[osgi-config-runmode-resolution.md](osgi-config-runmode-resolution.md).

---

## References

- [Run Modes (AEM 6.5, Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/implementing/deploying/configuring/configure-runmodes)
- [production-ready-mode-crxde.md](production-ready-mode-crxde.md)
- [osgi-config-runmode-resolution.md](osgi-config-runmode-resolution.md)
- [all-package-embed-structure.md](all-package-embed-structure.md)
