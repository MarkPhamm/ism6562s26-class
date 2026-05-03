# Week 2 — Docker & `docker-compose.yaml` Notes

Concepts and a key-by-key walkthrough of `week02/docker-compose.yaml`.

This week's stack is two services: **Postgres** (the database) and **pgAdmin** (a web UI for the database). Compose runs them together as one unit. Every concept below is shown alongside the actual line from the file.

---

## 1. What is Docker?

**Docker** is a tool that packages an application — together with its OS libraries, language runtime, and configuration — into a sealed unit called a **container**. The container runs the same way on any machine that has Docker installed, regardless of what's on the host.

Why people use it:

- **Same environment everywhere.** "Works on my laptop" stops being a problem — the same image runs identically on your laptop, a teammate's laptop, and production.
- **Isolation.** Each container has its own filesystem and processes. Postgres in one container can't see pgAdmin's files, and neither can mess with your host system.
- **Reproducibility.** A `docker-compose.yaml` is a checked-in recipe. Anyone who clones the repo and runs `docker compose up` gets the exact same Postgres + pgAdmin setup.
- **Cleanup.** `docker compose down` removes everything; nothing is left installed on your host.

### Image vs. container — the most important distinction

People mix these up constantly. They're not the same thing.

| | Image | Container |
| --- | --- | --- |
| What it is | A **read-only template** — a snapshot of a filesystem and metadata. | A **running instance** of an image. |
| Analogy | A class definition (or a recipe). | An object created from the class (or a meal cooked from the recipe). |
| How many | One image. | Many containers can run from the same image. |
| State | Immutable. Identified by a hash. | Has live state — running processes, writable filesystem layer, logs. |
| Created by | `docker build` or pulled from Docker Hub. | `docker run` / `docker compose up`. |

In `week02/docker-compose.yaml` the line `image: postgres:16` says **"build my container from the `postgres:16` image."** The image is downloaded from Docker Hub once; Compose then creates a fresh container from it.

### Compose vs. plain `docker run`

`docker run` starts one container at a time with command-line flags. **Docker Compose** is a thin layer on top: you write a YAML file describing one or more services and how they relate, then `docker compose up` brings them all up together with their networks, volumes, dependencies, and health checks wired correctly. The week 2 stack has two services that need to find each other — that's Compose's bread and butter.

---

## 2. The week 2 stack at a glance

```yaml
services:
  postgres:
    image: postgres:16
    container_name: postgres_db
    environment: { POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB }
    volumes:    [postgres_data → /var/lib/postgresql/data]
    ports:      [5432:5432]
    healthcheck: pg_isready
    networks:   [postgres_network]

  pgadmin:
    image: dpage/pgadmin4:8.13
    container_name: pgadmin
    environment: { PGADMIN_DEFAULT_EMAIL, PGADMIN_DEFAULT_PASSWORD }
    ports:       [8123:80]
    depends_on:  postgres (condition: service_healthy)
    networks:    [postgres_network]

networks:
  postgres_network: bridge

volumes:
  postgres_data:
```

```text
                     ┌──────────── your laptop (host) ────────────┐
                     │                                            │
   browser  ───────► │  localhost:8123 ──► pgAdmin container :80  │
                     │                          │                 │
   psql     ───────► │  localhost:5432 ────────┐│                 │
                     │                         ▼▼                 │
                     │             ┌── postgres_network (bridge) ─┐
                     │             │                              │
                     │             │   postgres:5432  ◄── pgadmin │
                     │             │   (talks by service name)    │
                     │             └──────────────────────────────┘
                     │                          │                 │
                     │                          ▼                 │
                     │                  postgres_data volume      │
                     │                  (survives `down`,         │
                     │                   wiped by `down -v`)      │
                     └────────────────────────────────────────────┘
```

Now we walk through every key.

---

## 3. `image:` — which image to run

```yaml
image: postgres:16
image: dpage/pgadmin4:8.13
```

Tells Compose *which image* this container should be created from.

- The part before the colon is the **image name** (`postgres`, `dpage/pgadmin4`).
- The part after the colon is the **tag** — usually a version (`16`, `8.13`). If you omit the tag you get `:latest`, which is risky because "latest" is a moving target.
- Images without a `/` (like `postgres`) are **official images** maintained by Docker. Images with a `/` (like `dpage/pgadmin4`) come from a publisher — `dpage` is the maintainer's Docker Hub username.

When you run `docker compose up`, Docker checks if the image is already on your machine. If not, it pulls it from Docker Hub. After that it's cached locally — subsequent `up`s start instantly.

**Pin your tags.** `postgres:16` is reproducible — six months from now the stack still works. `postgres:latest` may silently upgrade to Postgres 17 and break things.

---

## 4. `container_name:` — a friendly name for the container

```yaml
container_name: postgres_db
container_name: pgadmin
```

By default Compose names containers like `week02-postgres-1` (project + service + replica index). `container_name:` overrides that with a fixed name, so you can write:

```bash
docker logs postgres_db
docker exec -it postgres_db psql -U student -d mydb
```

instead of having to look up the auto-generated name.

**Trade-off:** because the name is fixed, you can't run two copies of the stack at the same time — the second one will collide on the name. For a class lab that's fine; for production-like setups people usually omit `container_name:` and let Compose auto-name.

---

## 5. `environment:` — config passed in at startup

```yaml
environment:
  POSTGRES_USER: student
  POSTGRES_PASSWORD: student
  POSTGRES_DB: mydb
```

Each line becomes an **environment variable** inside the container. Most official images are configured by env vars rather than config files because env vars are easy to set from a `docker-compose.yaml`, a CI system, or a cloud platform.

What these specific variables do (defined by the Postgres image's documentation, not by Docker):

| Variable | Effect on first container start |
| --- | --- |
| `POSTGRES_USER=student` | Creates a database superuser called `student`. |
| `POSTGRES_PASSWORD=student` | Sets that user's password. |
| `POSTGRES_DB=mydb` | Creates an empty database called `mydb`, owned by `student`. |

For pgAdmin:

| Variable | Effect |
| --- | --- |
| `PGADMIN_DEFAULT_EMAIL` | Login email for the pgAdmin web UI. |
| `PGADMIN_DEFAULT_PASSWORD` | Login password. |

**Important:** these only run on **first** container creation. If you change `POSTGRES_PASSWORD: student` to `POSTGRES_PASSWORD: hunter2` and restart, the password does **not** change — the database was already initialized with the old one. To pick up changes you need to delete the volume (`docker compose down -v`) and start fresh.

**Don't put real secrets in `environment:`** of a checked-in file. For class labs with throwaway passwords like `student`/`student` it's fine; for anything else use a `.env` file (which is `.gitignore`'d) or Docker secrets.

---

## 6. `volumes:` — persistent storage

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data    # service-level mount

volumes:                                       # top-level declaration
  postgres_data:
```

Two things called "volumes" in the same file. Don't get confused:

- The **top-level `volumes:`** block at the bottom *declares* a named volume called `postgres_data`. Docker manages the actual storage location on disk.
- The **service-level `volumes:`** list *mounts* that named volume into the container at the path `/var/lib/postgresql/data` (which is where Postgres stores all its data files).

### Why a volume is necessary

A container's filesystem is **ephemeral** — when the container is removed, anything written inside it is gone. Without a volume:

```text
docker compose up    → Postgres starts, creates tables, you load data
docker compose down  → container is destroyed → data is gone
docker compose up    → fresh empty database
```

With the named volume in place, the data files live *outside* the container, on a Docker-managed disk area. The container is disposable; the volume persists.

```text
docker compose up    → mounts volume → data is there
docker compose down  → container destroyed; volume kept
docker compose up    → mounts same volume → data is back
docker compose down -v  → -v ALSO deletes the volume → data is GONE
```

### Mount syntax

`SOURCE:TARGET[:MODE]`:

- `postgres_data:/var/lib/postgresql/data` — named volume mounted at the container path. Docker manages the source.
- `./local_dir:/data` — **bind mount** — a path on your host mounted into the container. Useful for "edit code on host, run inside container."
- Add `:ro` to make it read-only.

Week 2 uses the named-volume form. It's the right default for databases.

---

## 7. `ports:` — exposing the container to your host

```yaml
ports:
  - "5432:5432"   # Postgres
  - "8123:80"     # pgAdmin
```

Format is `HOST_PORT:CONTAINER_PORT`. **Read it as "map host port → container port".**

| Line | What happens |
| --- | --- |
| `"5432:5432"` | Postgres listens on port 5432 inside the container. Docker forwards `localhost:5432` on your laptop into the container. So `psql -h localhost -p 5432 ...` reaches Postgres. |
| `"8123:80"` | pgAdmin listens on port 80 inside the container (its default). Docker forwards `localhost:8123` on your laptop to that. So <http://localhost:8123> opens pgAdmin. The host port doesn't have to match the container port. |

### Why pick a non-default host port (8123)?

Many laptops already have something on port 80 (a local web server, dev tool, etc.). Picking 8123 avoids a collision. The container still thinks it's serving on port 80; only the host-side mapping is different.

### Without `ports:`

The service is reachable **from other containers on the same network** but **not** from your laptop's browser or terminal. That's actually fine for many services — Postgres in production is often *only* exposed to its app containers, not to the public internet.

---

## 8. `healthcheck:` — is the service actually ready?

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U student"]
  interval: 10s
  timeout: 5s
  retries: 5
```

A container can be **running** but not yet **ready** to accept connections. Postgres takes a few seconds after startup to initialize the data directory, accept users, etc. A healthcheck is a small command Docker runs **inside the container** repeatedly, and uses to label the container as `starting`, `healthy`, or `unhealthy`.

| Field | Meaning |
| --- | --- |
| `test` | The command to run. `CMD-SHELL` means "run this string in a shell." `pg_isready` is a Postgres CLI tool that returns exit code 0 when the server is accepting connections, non-zero otherwise. |
| `interval: 10s` | Wait 10 seconds between checks. |
| `timeout: 5s` | If a single check takes longer than 5s, treat it as a failure. |
| `retries: 5` | Mark the container `unhealthy` only after 5 consecutive failures (so a single hiccup doesn't trip an alarm). |

You can see the status with:

```bash
docker ps
# look at the STATUS column: "Up 12s (healthy)" or "Up 3s (health: starting)"
```

Healthchecks become really useful when paired with `depends_on: condition: service_healthy` — see next section.

---

## 9. `networks:` — how containers find each other

```yaml
networks:
  postgres_network:           # top-level: declare the network
    driver: bridge

services:
  postgres:
    networks:                 # service-level: attach to it
      - postgres_network
  pgadmin:
    networks:
      - postgres_network
```

By default, Compose creates a private network and puts every service on it. Inside that network, services can reach each other **by service name** as a DNS hostname. So in pgAdmin, when you add a server connection, you point it at `postgres` on port `5432` — *not* `localhost:5432`.

```text
   pgadmin container   ──connects to──►   postgres:5432
                                           ▲
                                           │  Compose's built-in DNS resolves
                                           │  "postgres" → the postgres container's IP
```

`localhost` from inside a container means *that container itself*, not your laptop. That's a classic stumbling point — pgAdmin trying to connect to `localhost:5432` would be looking for Postgres *inside the pgAdmin container*, which obviously isn't there.

### `driver: bridge`

`bridge` is the default driver for single-host setups: a virtual switch on your machine that the participating containers attach to. Other drivers (`host`, `overlay`) exist for advanced cases — you don't need them this week.

### Why declare a network at all?

If you omit `networks:` entirely, Compose creates a default network for you and you'd still be fine. Declaring `postgres_network` explicitly is just clearer about *what* connects to *what*, and it scales well — in a bigger stack you might give the database one network and the public-facing services another.

---

## 10. `depends_on:` — startup ordering

```yaml
pgadmin:
  depends_on:
    postgres:
      condition: service_healthy
```

Says: **don't start `pgadmin` until `postgres` is healthy.** Without this, both containers would launch at the same instant, and pgAdmin might try to register a Postgres connection while Postgres is still initializing — leading to a confusing failure on first startup.

### Two flavors of `depends_on`

There's a short form and a long form:

```yaml
# Short form — wait until the dependency has STARTED (process is up).
# Doesn't say anything about whether it's actually serving traffic.
depends_on:
  - postgres

# Long form — wait for a specific condition.
depends_on:
  postgres:
    condition: service_healthy   # wait for the healthcheck to pass
    # other options: service_started (default short-form behavior)
    #                service_completed_successfully (for one-shot tasks)
```

The long form is what you want when the dependency takes time to initialize (databases, message brokers, search engines). It pairs with the `healthcheck:` from the previous section: pgAdmin won't start until `pg_isready` returns success.

### What `depends_on` does *not* do

- It does **not** restart `pgadmin` if `postgres` later crashes — it only controls **startup order**.
- It does **not** make pgAdmin reconnect on its own if the database briefly goes away. Reconnection logic is the application's job.

---

## 11. Common commands cheatsheet

```bash
docker compose up -d              # start the stack in the background
docker compose ps                 # see what's running and their health
docker compose logs -f postgres   # tail logs from one service
docker compose exec postgres psql -U student -d mydb   # shell into a running container

docker compose down               # stop and remove containers (keeps volumes)
docker compose down -v            # ALSO delete the named volumes — destructive

docker compose pull               # refresh images to latest matching tag
docker compose up -d --build      # rebuild local images and restart
```

---

## Week 2 Takeaways

1. **Image vs. container.** Image = read-only template. Container = running instance. One image, many containers.
2. **`image:` pins what you run.** Always tag a version; `latest` is a moving target.
3. **`container_name:`** is a convenience — easy `docker exec` / `docker logs` — at the cost of preventing parallel stacks.
4. **`environment:`** configures the image at first start. Changes don't take effect until you rebuild the volume for stateful services.
5. **`volumes:`** keep data alive across `docker compose down`. Without one, your database is reset every restart.
6. **`ports:`** expose container ports to your laptop. Inside the network, containers don't need ports — they reach each other by service name.
7. **`healthcheck:`** distinguishes "process is alive" from "service is actually ready."
8. **`networks:`** give services DNS-by-name. `localhost` inside a container ≠ your laptop.
9. **`depends_on: condition: service_healthy`** is what makes pgAdmin wait for Postgres to *actually* be ready, not just *started*.
