# In-place upgrade to AEM 6.5: the parts that are not obvious

The Adobe procedure is mostly mechanical. This keeps only the steps where the
obvious action is the wrong one, plus the decision table for whether you need a
repository migration at all.

**Sourcing:** Adobe, *In-Place Upgrade* (AEM 6.5), read September 2026. Not
performed against a real instance — treat the commands as the documented shape,
not as reproduced output. The page defers pre-upgrade maintenance, code review,
post-upgrade checks, backup targets, downtime and rollback to separate
documents; none of that is covered here.

> **The page's Java guidance is stale — ignore it.** It states a Java 7 minimum
> and "Oracle JRE 8 or IBM JRE 7 & 8 only" for 6.3+. For what 6.5 and 6.5 LTS
> actually run on, see
> [aem-sdk-java-ceiling-dual-65-cloud.md](aem-sdk-java-ceiling-dual-65-cloud.md)
> and [java11-bootdelegation-drift.md](java11-bootdelegation-drift.md) (Java 11
> on 6.5, JDK 21 on LTS, and the `sling.properties` bootdelegation drift that
> bites when you move an existing instance onto a newer JVM).

---

## Start from the jar, not the start script

The one instruction most likely to be skipped, because every other day of that
instance's life you use `bin/start`:

> Start AEM using the **jar file directly**, not the start scripts.

Only during the upgrade. Afterwards the start script is fine again.

The problem this creates: you now need the instance's real start command, and
on anything long-lived that is not what the runbook says it is. Recover it from
the running process **before you stop it**:

```shell
ps -ef | grep java
```

Then substitute the jar and drop the launchpad-era arguments, keeping every
JVM parameter and run mode verbatim:

```shell
# before — as the start script invoked it
/usr/bin/java -server -Xmx1024m -Djava.awt.headless=true \
  -Dsling.run.modes=author,crx3,crx3tar \
  -jar crx-quickstart/app/cq-quickstart-6.5.0-standalone-quickstart.jar \
  start -c crx-quickstart -i launchpad -p 4502 \
  -Dsling.properties=conf/sling.properties

# after — new jar, sibling of crx-quickstart
/usr/bin/java -server -Xmx1024m -Djava.awt.headless=true \
  -Dsling.run.modes=author,crx3,crx3tar \
  -jar cq-quickstart-6.5.0.jar \
  -c crx-quickstart -p 4502 \
  -Dsling.properties=conf/sling.properties
```

Three changes worth noticing: the jar moves from
`crx-quickstart/app/…-standalone-quickstart.jar` to a **new jar placed beside
`crx-quickstart`**, the `start` subcommand and `-i launchpad` disappear, and
`-Dsling.run.modes` is carried across untouched. Losing a run mode here
silently changes which `config.<runmode>` folders resolve on the upgraded
instance — see
[osgi-config-runmode-resolution.md](osgi-config-runmode-resolution.md) for what
that costs.

The new jar is unpacked first, outside `crx-quickstart`:

```shell
java -Xmx4096m -jar aem-quickstart.jar -unpack
```

---

## Do you need a repository migration at all?

Usually not. This is the table to check before reading anything about crx2oak:

| Source | Migration needed? |
|---|---|
| AEM **6.3+**, TarMK | **No** |
| AEM **MongoMK**, any version | **No** |
| AEM **older than 6.3**, TarMK | **Yes** |
| AEM **older than 5.6** | Yes — and via 6.0 first; there is no direct path to 6.5 |

If you land in the "no" rows, the crx2oak material is irrelevant to you and the
upgrade is jar-swap plus code redeploy.

### If you do need it

```shell
java -Xmx4096m -jar aem-quickstart.jar -v -x crx2oak -xargs -- \
  --load-profile <PROFILE> <FLAGS>
```

| Source | Target | Profile |
|---|---|---|
| crx2/TarMK + FileDataStore | TarMK | `segment-fds` |
| TarMK/crx2 + S3DataStore | TarMK | `segment-custom-ds` |
| TarMK, no datastore | TarMK | `segment-no-ds` |
| crx2 | MongoMK | `mongo-from-crx2` (+ `-T mongo-uri=… -T mongo-db=…`) |

**The bundled crx2oak is not the current one.** The instance ships
`crx-quickstart/opt/extensions/crx2oak.jar`, and Adobe points you at Maven
Central for the latest build instead. Using the shipped jar is the default
action and the wrong one.

If migration cannot reach the binaries, it needs to be told where they are —
`--src-datastore=/path/to/datastore` for a file datastore, or
`--src-s3config=/path/to/SharedS3DataStore.config` plus
`--src-s3datastore=/path/to/datastore` for S3. On Windows, `--disable-mmap`
works around Java memory-mapping failures.

---

## Don't bundle the datastore migration into the upgrade

FileDataStore became the default at 6.3, but **an external datastore is not
required**, and Adobe's recommendation is explicit: upgrade **without**
migrating the datastore, then migrate separately afterwards if you want to.

Worth obeying because it keeps the failure domains apart. An upgrade that also
moves every binary has two independent ways to fail and one rollback story
covering both.

---

## Verifying

- **Exit code zero** from the migration.
- `crx-quickstart/logs/upgrade.log` — read it for `WARN` and `ERROR` even on a
  zero exit; the exit code is necessary, not sufficient.
- Confirm the configuration files under `crx-quickstart/install/` were updated.

---

## The run-mode promotion flag

`--promote-runmode nosamplecontent` is available during a **TarMK crx2oak
migration**, and removes sample content as part of the migration.

This is the one documented route to `nosamplecontent` on an instance that did
not start life with it — a qualification on the otherwise flat rule that
installation run modes are frozen
([production-ready-mode-crxde.md](production-ready-mode-crxde.md),
[runmode-sources-and-precedence.md](runmode-sources-and-precedence.md)).

Read it narrowly, though. crx2oak is rebuilding the repository, so this is
closer to "you have a new repository" than to "you flipped a frozen mode in
place," and it is unavailable on a 6.3+ source, where no migration runs at all.
It does not give you a way to harden a running instance. Documented, not
reproduced.

---

## References

- [In-Place Upgrade (AEM 6.5, Adobe docs)](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/implementing/deploying/upgrading/in-place-upgrade)
- [production-ready-mode-crxde.md](production-ready-mode-crxde.md)
- [runmode-sources-and-precedence.md](runmode-sources-and-precedence.md)
- [osgi-config-runmode-resolution.md](osgi-config-runmode-resolution.md)
- [revision-cleanup.md](revision-cleanup.md)
