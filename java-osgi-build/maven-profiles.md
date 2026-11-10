# Maven profiles: defining, activating and checking them

A Maven profile is a named block of build settings that applies only when it is active. One `pom.xml` can then build differently for local dev, CI and production without being edited.

## Defining a profile

```xml
<profiles>
  <profile>
    <id>dev</id>
    <properties>
      <env.url>http://localhost:8080</env.url>
      <skipTests>true</skipTests>
    </properties>
  </profile>

  <profile>
    <id>prod</id>
    <properties>
      <env.url>https://example.net</env.url>
    </properties>
    <build>
      <plugins>
        <!-- plugin that runs only in prod builds -->
      </plugins>
    </build>
  </profile>
</profiles>
```

A profile can override almost anything the main POM declares: properties, dependencies, `dependencyManagement`, plugins, `modules`, repositories, distribution management. Its content is merged on top of the main POM when it is active.

## Activating a profile

| Trigger | Example |
|---|---|
| Command line | `mvn clean install -P dev` |
| Several at once | `mvn install -P dev,fast` |
| Explicitly off | `mvn install -P '!dev'` (quote the `!` in bash and zsh) |
| On by default | `<activation><activeByDefault>true</activeByDefault></activation>` |
| System property present | `<activation><property><name>ci</name></property></activation>`, then `mvn install -Dci` |
| System property has a value | `<property><name>env</name><value>prod</value></property>`, then `-Denv=prod` |
| Environment variable | `<property><name>env.CI</name></property>` (any env var, `env.` prefix) |
| JDK, OS, file | `<jdk>17</jdk>`, `<os><family>windows</family></os>`, `<file><exists>${basedir}/local.flag</exists></file>` |

**Gotcha: `activeByDefault` is fragile.** It turns off as soon as any other profile in the same POM is activated with `-P` or by a trigger. "Default" means "only when nothing else is active". To keep a profile always on, activate it with a property that is always set, or with `!` negation (`<property><name>!skipDefault</name></property>` is active unless `-DskipDefault` is passed).

## Checking what is active

```
mvn help:active-profiles
mvn help:effective-pom -P prod
```

`effective-pom` prints the final merged POM, which is the quickest way to see what a profile actually changes.

## Where profiles live

- **`pom.xml`**: committed and shared with the team. Use this for nearly everything.
- **`~/.m2/settings.xml`**: per machine, applies to every project. Good for mirrors, extra repositories and local-only properties. Activate via `<activeProfiles>` in the same file. Credentials go in `<servers>` there, never in a POM.
- Profiles in a parent POM are inherited by modules. A profile activated on the command line applies to every module in the reactor that defines it.

## Typical uses

- Deploy targets that differ per environment (URLs, ports, credentials looked up from `settings.xml`).
- Fast local builds that skip tests, code-quality plugins or frontend builds.
- Extra modules that build only in some situations (`<modules>` inside the profile).
- Different dependency versions per target platform.

## What this means for AEM

- **Archetype deploy profiles.** Projects generated from the AEM Project Archetype ship profiles that install packages into a running local instance through the Content Package Maven Plugin: `autoInstallSinglePackage` (the `all` package to author on `localhost:4502`), `autoInstallSinglePackagePublish` (same, publish on `4503`), `autoInstallPackage` / `autoInstallPackagePublish` (each content package separately), and `autoInstallBundle` (just the `core` bundle). Typical dev loop: `mvn clean install -PautoInstallSinglePackage`, or `mvn clean install -PautoInstallBundle` inside `core/` for a Java-only change. Host, port and credentials are properties (`aem.host`, `aem.port`, `vault.user`, `vault.password`) you can override with `-D`.
- **Cloud Manager can't take `-P`.** The pipeline runs a fixed Maven command, so a profile meant only for Cloud Manager builds must activate on an environment variable Cloud Manager sets, such as `CM_BUILD`:

  ```xml
  <activation><property><name>env.CM_BUILD</name></property></activation>
  ```

  For the reverse case (a profile that must not run in Cloud Manager), use `<name>!env.CM_BUILD</name>`. Docs-based: Cloud Manager's build environment documentation lists the variables it sets (`CM_BUILD`, `BRANCH`, `CM_PROGRAM_ID`, `CM_ENVIRONMENT_NAME` and others).
- **Don't hide required build steps in a local-only profile.** If frontend compilation or package validation runs only under a profile, Cloud Manager's build won't run it and the deployed output will differ from local.
