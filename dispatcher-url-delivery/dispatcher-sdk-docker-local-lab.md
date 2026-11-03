# Running the AEMaaCS SDK Dispatcher locally in Docker (a dispatcher lab repo)

The Dispatcher Tools inside the AEMaaCS SDK zip ship the real Apache + Dispatcher image plus run scripts, and those scripts take **any folder** as the config source. That is what makes a standalone "dispatcher lab" repo possible: the config does not have to live inside the Maven project, and you can experiment against a local publish without touching the project until a change is worth keeping. Confirmed on Dispatcher Tools 2.0.x (SDK 2026.x); the script interface has been stable since 2.0.

Background on what the dispatcher does is in [[dispatcher-fundamentals]]; the cache rules and their log lines are in [[dispatcher-caching-and-invalidation]] and [[dispatcher-cacheability-rules-and-log-messages]]; the SDK itself is in [[aemaacs-sdk]].

---

## What is in the SDK zip

| File | What it is |
|---|---|
| `aem-sdk-quickstart-<version>.jar` | the AEM runtime, author *or* publish depending on the filename it is started under |
| `aem-sdk-dispatcher-tools-<version>-unix.sh` | **self-extracting archive**: run it and it unpacks `dispatcher-sdk-<version>/` into the current directory |
| `aem-sdk-dispatcher-tools-<version>-windows.zip` | same content for Windows |

The unpacked tools folder:

```
dispatcher-sdk-<version>/
├── bin/
│   ├── docker_run.sh             # run the container, or `test` mode
│   ├── docker_run_hot_reload.sh  # same, re-reads config on every file change
│   ├── validate.sh               # the Cloud Manager validator + httpd -t in the image
│   ├── docker_immutability_check.sh
│   └── validator                 # the binary validate.sh wraps
├── lib/
│   ├── dispatcher-publish-amd64.tar.gz   # the Docker image, loaded on first run
│   └── dispatcher-publish-arm64.tar.gz
└── docs/
```

Keep the tools version paired with the quickstart you run: take both from the **same SDK zip**. An older tools build unpacked months ago is not the one that matches today's quickstart.

## Lab repo layout

```
dispatcher-lab/
├── .gitignore          # sdk/, cache/, .idea/, .DS_Store
├── README.md           # which SDK version, the three commands
├── src/                # copy of <project>/dispatcher/src — committed untouched as the baseline first
│   ├── conf.d/
│   ├── conf.dispatcher.d/
│   └── opt-in/USE_SOURCES_DIRECTLY
├── scripts/            # run.sh, test.sh, validate.sh — one-line wrappers, see below
├── sdk/                # gitignored: the unpacked Dispatcher Tools
└── cache/              # gitignored: the container's document root, i.e. what got cached
```

- `src/` is a plain copy of the archetype's `dispatcher/src`. The two symlinks (`enabled_vhosts/default.vhost`, `enabled_farms/default.farm`) survive `cp -R` and Git, and resolve inside the bind mount.
- `opt-in/USE_SOURCES_DIRECTLY` switches the scripts to **flexible mode**: the whole folder is mounted read-only at `/mnt/dev/src` in the container. Without the marker the script mounts only a fixed list of files one by one.
- Commit `src/` **before** the first experiment so every later diff is exactly the experiment.
- The wrappers exist only to pin arguments and to `cd` to the repo root first: the SDK scripts resolve the config folder and write `cache/` **relative to the current directory**.

```bash
#!/bin/bash
cd "$(dirname "$0")/.." && sdk/bin/docker_run_hot_reload.sh "$PWD/src" host.docker.internal:4503 8080
```

`host.docker.internal` is how the container reaches a publish running on the host.

## The three ways to run it

| Command | What happens | Use it for |
|---|---|---|
| `docker_run.sh <src> host.docker.internal:4503 8080` | Apache + Dispatcher on `localhost:8080` in front of publish | serving |
| `docker_run_hot_reload.sh …` (= `HOT_RELOAD=true docker_run.sh …`) | same, config re-read on every save | day-to-day experiments: edit, refresh |
| `docker_run.sh <src> localhost:4503 test` | runs `httpd -t` and **prints the fully resolved `dispatcher.any`** with every include and variable expanded, then exits | "what did the dispatcher actually load?" |
| `validate.sh <src>` | phase 1: the validator binary over the `.any` files; phase 2: `httpd -t` in the image | before copying a config back into the project: this is what Cloud Manager's build runs |

Environment variables on any of them:

| Variable | Values | Effect |
|---|---|---|
| `DISP_RUN_MODE` | `dev` (default), `stage`, `prod` | simulates the environment type |
| `DISP_LOG_LEVEL` | `trace1`, `debug`, `info`, `warn` (default), `error` | dispatcher log; `trace1` shows traffic to the backend |
| `REWRITE_LOG_LEVEL` | `trace1`–`trace8`, `debug`, …, `warn` (default) | `mod_rewrite` log; `trace2` shows which rules fire |
| `HTTPD_DUMP_VHOSTS` | `true`/`false` | dumps the vhosts for debugging |
| `ALLOW_CACHE_INVALIDATION_GLOBALLY` | `true`/`false` | overrides `default_invalidate.any` so any client may flush |
| `ENV_FILE` | path | a file of env vars imported into the container |

The container runs in the foreground; leave the terminal open. First run loads the image from `lib/`, which takes a minute, and needs the Docker daemon up: `validate.sh` phase 2 and every run fail with "failed to connect to the docker API" otherwise.

## What you may edit

Only the customer files: `rewrite.rules`, `custom.vars`, `global.vars`, `filters.any`, `rules.any`, `clientheaders.any`, `virtualhosts.any`, `*.farm`, `*.vhost`. Everything prefixed `default_` plus `dispatcher_vhost.conf` is immutable on the cloud, and `docker_immutability_check.sh` (also run by the validator) flags edits to them.

The archetype's root rewrite builds its target from a variable:

```apache
# conf.d/variables/custom.vars
Define CONTENT_FOLDER_NAME <project>
# conf.d/rewrites/rewrite.rules
RewriteRule ^/?$ /content/${CONTENT_FOLDER_NAME}/us/en.html [PT,L]
```

Change the variable to switch site folder in both the root rule and the catch-all that prefixes `/content/<folder>/` onto every `.html` request; edit the rule itself to change the landing page. Keep `[PT,L]` so the dispatcher caches under the rewritten path ([[url-resolution-layers-aemaacs]]).

## Reading the first response

On a publish with no content, `http://localhost:8080/` returning **404 from AEM is a success**: the request went through Apache, passed the filters and reached publish. Other codes:

| Code | Meaning |
|---|---|
| 403 | a `/filter` rule blocked it: `Filter rejects` in the dispatcher log at Info/Debug ([[dispatcher-common-log-entries]]) |
| 502 / 503 | the container cannot reach the backend, or publish is too slow: check `host.docker.internal:4503` and `/renders` → `/timeout` |
| 404 with Adobe's page | chain OK, nothing at that path on publish |

## The lab publish

The SDK runs author or publish from the **same jar**; the filename decides. A publish next to an existing author is a second folder with a copy of the jar renamed `aem-publish-p4503.jar` (or `cq-publish-p4503.jar`), started with its own run modes. Two traps:

- **`-r` replaces the run modes derived from the filename.** `-r author,local` on a jar named `*-publish-*` installs an **author** on 4503, and `config.publish` OSGi configs never load. The tier is fixed on first start; delete `crx-quickstart` to change it ([[local-development-environment]]).
- **A second JDWP agent on the same port kills the JVM at start** ("transport error 202: bind failed"). Give the publish a different debug port or none.

Populate it from author with the legacy replication agent the local jar still ships (`/etc/replication/agents.author.html` → Default Agent → transport `http://localhost:4503/bin/receive?sling:authRequestLogin=1`). Log in as admin on publish through `/libs/granite/core/content/login.html`; publish has no login link of its own. Pointing the container at **author** instead is a poor target: every author request carries a `login-token` cookie, which the dispatcher treats as authenticated and never caches ([[dispatcher-permission-sensitive-caching]]), and the default filters block the Touch UI.

## Short URLs on a publish nothing deploys to

A lab publish gets no packages, so a convenience URL such as `/publish/login` has to be a hand edit. Use the mapping tree rather than `/libs`:

```
/etc/map/http/localhost.4503/publish/login     (sling:Mapping)
    sling:match    = login/?$
    sling:redirect = /libs/granite/core/content/login.html
    sling:status   = 302 (Long)
```

- One node named `localhost.4503`: the dot is literal, the key is `<scheme>/<host>.<port>`.
- An **external** redirect, not `sling:internalRedirect`: the login page links its favicon relatively and posts to its real path.
- `sling:match` is needed because a node-name entry registers with a **trailing slash** and only matches `/publish/login/`; and its value must **not** start with `/`, or the pattern gets a double slash. Details in [[sling-resource-resolution-mapping]].

**Sling POST servlet trap when hand-editing a node that carries an ACL**: `jcr:mixinTypes` in a POST **replaces** the whole mixin list. Sending only `sling:VanityPath` to a node that had `rep:AccessControllable` drops the mixin, which deletes its `rep:policy`; on a `/libs` node whose only `everyone` read grant lived there, anonymous requests turn into 404 with no error logged. Always resend every existing mixin. Recovery for a `/libs` node: reinstall the Adobe content package that owns it from `/etc/packages` (Package Manager keeps them; for the login page it is `com.adobe.granite.platform.login`), which rewrites the node and its policy exactly.

## What this means for AEM

- **Cloud Manager deploys only the project's `dispatcher` module.** The lab is for experiments; a result is copied back into `<project>/dispatcher/src` and must pass `validate.sh` there. The lab gives you what no cloud tier does: the resolved `dispatcher.any`, rewrite traces, and a browsable cache folder ([[local-development-environment]]).
- The image is the **publish** flavour and the cloud's immutability rules apply inside it. For unconstrained 6.5-style experiments you would need a plain `httpd:2.4` image plus the **Linux** dispatcher module; the macOS module build cannot run in a container.
- On AEMaaCS there is no customer dispatcher in front of author, so author-side behaviour observed in the lab has no cloud counterpart.
