# Docker Compose: what it is, the file, the commands, the traps

## What it is

Docker runs one container per `docker run` command. **Docker Compose** describes several containers in one YAML file (`docker-compose.yaml` or `compose.yaml`) and starts or stops them together with one command. Think of it as a recipe for "everything this app needs to run locally", written down once instead of retyped as long `docker run` lines.

Compose also creates a **private network** for the project. Every service is reachable from the others by its **service name** as a hostname (`azurite`, `db`), so containers never need IP addresses.

## The file

```yaml
name: my-project              # groups containers + network; `down` removes exactly these

services:
  <service-name>:             # also the hostname other services use
    image: <image>:<tag>      # what to run
    container_name: <name>    # optional fixed name for `docker logs <name>`
    ports:
      - "8080:80"             # "<port on your machine>:<port in container>"
    environment:
      SOME_SETTING: value     # quote "Y", "yes", "on": bare ones are YAML booleans
    volumes:
      - ./local.json:/path/in/container/file.json   # bind a local file/folder in
    depends_on:
      - <other-service>       # start order only, see traps
```

Most services need only `image` plus a few of `ports`, `environment` and `volumes`.

### Worked example: Azure Event Hubs emulator + Azurite

```yaml
name: event-hubs-local
services:
  emulator:
    container_name: eventhubs-emulator
    image: mcr.microsoft.com/azure-messaging/eventhubs-emulator:latest
    volumes:
      - ./Config.json:/Eventhubs_Emulator/ConfigFiles/Config.json   # namespace, hubs, partitions, consumer groups
    ports:
      - "5672:5672"   # AMQP (SDK clients)
      - "9092:9092"   # Kafka endpoint
      - "5300:5300"   # health
    environment:
      BLOB_SERVER: azurite        # service name used as hostname
      METADATA_SERVER: azurite
      ACCEPT_EULA: "Y"
    depends_on:
      - azurite
  azurite:
    container_name: azurite
    image: mcr.microsoft.com/azure-storage/azurite:latest
    ports:
      - "10000:10000"   # blob
      - "10001:10001"   # queue
      - "10002:10002"   # table
```

The emulator finds storage at `azurite` over the project network, while your code on the host connects to `localhost:5672`. Two different addresses for two different callers. See [[event-hubs]].

## The commands

Run them from the folder that contains the file, or pass `-f <file>`.

| Command | What it does |
|---|---|
| `docker compose up -d` | Create and start everything in the background (`-d` = detached) |
| `docker compose ps` | Show the project's containers and their state |
| `docker compose logs -f <service>` | Follow one service's log |
| `docker compose stop` / `start` | Stop / restart the containers, keep them |
| `docker compose down` | Stop and **remove** the containers and the network |
| `docker compose down -v` | Same, plus the named volumes (**data gone**) |
| `docker compose pull` | Fetch newer images for `:latest` tags |
| `docker compose config` | Print the resolved file; `-q` = validate only |

After editing the file, run `docker compose up -d` again. Compose recreates only the services whose configuration changed.

## Traps

- **`depends_on` orders start-up, it does not wait for readiness.** The dependent container can start before its dependency accepts connections. Either retry (`up -d` again), or add a `healthcheck` to the dependency and use `depends_on: { db: { condition: service_healthy } }`.
- **`localhost` means different things.** Inside a container it is the container itself. Containers talk to each other by service name. Code on the host uses `localhost:<published port>`. A container that needs the host uses `host.docker.internal` (Docker Desktop).
- **Port order is host first.** `"8081:80"` publishes container port 80 on host port 8081. "Port already in use" refers to the left number.
- **`:latest` is not pinned.** `up` reuses the cached image. Run `pull` to update, or pin a tag for reproducibility.
- **`down` vs `stop`.** `stop` keeps container state, while `down` deletes containers (and with `-v`, volumes). A bind-mounted local file survives both.
- **`container_name` is global.** Two projects using the same fixed name collide, and a service with `container_name` cannot be scaled.
- **YAML booleans.** `ACCEPT_EULA: Y` without quotes may be parsed as `true`. Quote such values.
- **An empty `services:` key fails validation** ("services must be a mapping"). Every key needs content before `config` or `up` accepts the file.

## What this means for AEM

Compose is not part of running AEM itself. The AEM SDK quickstart is a jar, and the AEMaaCS Dispatcher SDK ships its own `docker_run.sh` scripts instead of a compose file (see [[dispatcher-sdk-docker-local-lab]]). In the archetype's `ui.tests` module, the `Dockerfile` and compose files are protected contract files that Cloud Manager relies on, so don't restructure them (see [[cloud-manager-functional-ui-testing]]). Where Compose earns its place is in the systems around AEM in a local setup: emulators for Azure messaging and storage, a database or search engine behind an integration, a mock external API. All of them start with one `up -d` next to the local author and publish.
