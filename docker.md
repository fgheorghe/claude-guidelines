# Container conventions

Guidance for writing Dockerfiles and `docker-compose.yml` files in this project. The rules below are defaults. There are good reasons to deviate, but deviation should be deliberate and the reason should be obvious from a comment in the affected file.

The project uses one Dockerfile and one compose file for both local development and production. Where dev and prod conflict, the production-safe option wins and dev convenience is added as an additive stage, an additive compose service, or an opt-in via `.env`.

The dev runtime is typically Podman; production is Docker Engine. The compose file is written to work under both — see "Runtime split" near the end for the small set of differences that matter.

## Principles

These rules are referenced throughout the rest of this guide. Each has a short name; later sections refer to principles by name instead of restating the reasoning. Read these once; they make the rest of the document terse.

**Production by default.** Every variable, every default, every behavior is production-safe out of the box. A bare `docker compose up` on a server with no `.env` produces a hardened, localhost-only, production-shaped stack. Dev is opt-in via `.env`.

**Compose is deployment, code is application.** Compose owns when containers start, what images they use, what volumes they have, what env vars are set. Application code owns the schedule, the logic, the locking, the error handling. The boundary is sharp: no application logic in compose `command:` blocks, no compose concerns in application code.

**Data and artifacts split.** Persistent data (databases, search indexes, user uploads, generated reports, logs) is bind-mounted to a known host path. Derived artifacts (`node_modules`, build caches) are named volumes scoped to a service. The test: "could I regenerate this from source?" Yes means named volume. No means bind mount with documented ownership.

**`.env` is infrastructure; `config.ini` is application.** `.env` holds what compose needs to assemble the stack (project name, ports, host paths, uids, build target). `config.ini` holds what the app needs to behave (feature flags, cache TTLs, third-party endpoints). When adding a new configurable, ask which side needs it; that determines where it goes.

**Inline documentation beats external docs.** Anything a fresh checkout needs to do before `docker compose up` works — chown, mkdir, secrets to generate — is documented as a comment in the compose file directly above the relevant line. External docs drift; inline comments do not.

**Production never copies dev, dev never bakes production.** Production images are self-contained: source baked in, dependencies installed, ready to run. Dev images bind-mount source, mount config, mount data, reload on changes. The dev stage and the runtime stage are different targets in the same multi-stage Dockerfile.

**No copies of secrets.** `.env` is never copied into a container. `config.ini` is never copied into a Dockerfile. Secrets reach the running process via compose `environment:` substitution or read-only bind mounts. Anything copied into an image is in its history forever.

**Assume compromise.** The threat model assumes a dependency or the app itself will be compromised at some point. Hardening minimizes what an attacker can do after that point: no shell, no extra binaries, no writable filesystem, no network reach beyond what the app needs, no host access beyond what is strictly required.

**Friction over magic.** Where a decision has security implications (volume ownership, network exposure, privileged access), prefer explicit configuration that requires a conscious choice over automatic behavior that papers over the question. A chown step before first run is friction; that friction is the point.

**`COMPOSE_PROJECT_NAME` is an instance namespace, not a mode flag.** It exists so multiple instances of the same product can run on one host without colliding on container names, image tags, volumes, or networks. Examples: `steam-proxy-main`, `steam-proxy-feature-search`, `steam-proxy-prod-blue`. It never decides behavior; `APP_MODE` decides mode.

The rest of the file applies these principles to specific concerns. When a rule cites a principle by name, that is the load-bearing reason; the surrounding text adds specifics rather than restating the case.

## Environment variable naming, `.env`, and `config.ini`

Three families of variables show up in compose, distinguished by scope and prefix.

**Compose-recognized variables.** Names Docker Compose itself reads. Fixed by Docker, not chosen by us: `COMPOSE_PROJECT_NAME`, `COMPOSE_FILE`, `COMPOSE_PROFILES`. Set in `.env` or the shell as needed.

**Project-wide variables.** Things that affect every service in the stack. No prefix, SCREAMING_SNAKE_CASE: `APP_MODE`, `BUILD_TARGET`, `NODE_ENV`, `BIND_IP`, `TZ`, `UID`, `GID`, plus volume host paths like `DATA_PATH`, `LOGS_PATH`, `CACHE_PATH`.

**Service-specific variables.** Things that affect one service. Prefix with the service role from `<project>-<role>` naming. For `steam-proxy-web`: `WEB_HOST_PORT`. For `steam-proxy-cron`: `CRON_LOG_LEVEL`. For state services, use a short role abbreviation: `DB_*`, `MEILI_*`, `REDIS_*`, `CACHE_*`, `QUEUE_*`.

A complete `.env` reads top-to-bottom by scope: compose-recognized first, then project-wide, then service-specific grouped by service.

```bash
# Compose
COMPOSE_PROJECT_NAME=steam-proxy-main

# Project-wide
APP_MODE=dev
BUILD_TARGET=dev
NODE_ENV=development
BIND_IP=127.0.0.1
TZ=Europe/London

# Web service
WEB_HOST_PORT=3000

# Database
DB_HOST_PORT=5432
DB_PASSWORD=...

# Meilisearch
MEILI_HOST_PORT=7700
MEILI_MASTER_KEY=...
```

### What goes in `.env` vs. `config.ini`

By **`.env` is infrastructure; `config.ini` is application**, the question for any new configurable is: does compose need to route, mount, or expose this value? If yes, it belongs in `.env`. If only the application reads it, it belongs in `config.ini`.

Examples of the split. In `.env` (compose needs these to assemble the stack):

```bash
WEB_HOST_PORT=3000
DATA_PATH=./data
MEILI_MASTER_KEY=...
DB_PASSWORD=...
```

In `config.ini` (only the app needs these):

```ini
[search]
; The maximum number of results returned per search query.
max_results = 50
; The default ranking algorithm. One of: relevance, popularity, recency.
ranking = relevance

[cache]
; The TTL for cached search results, in seconds.
ttl_seconds = 300

[features]
; Whether to enable the experimental price-tracking UI.
price_tracking = false
```

Compose has no reason to know about `ranking` or `ttl_seconds`. Keep them out of `.env` so `.env` stays focused and readable.

### `config.ini` is bind-mounted, never copied

By **No copies of secrets** and **Production never copies dev**, `config.ini` is always bind-mounted read-only:

```yaml
volumes:
  - ${CONFIG_PATH:-./config.ini}:/app/config.ini:ro
```

Never `COPY config.ini` in the Dockerfile. The configurable host path means production points at a system path (`/etc/<app>/config.ini`) while dev uses the repo-local version, both with the same image. Changing config does not require a rebuild. Secrets stored in the config file never enter image layers and never appear in `docker history`.

### `.env` is never copied into a container

`.env` is read by compose at startup. Compose substitutes its values into the YAML and passes them to containers via `environment:`. The container itself never reads the file.

Do not `COPY .env` in a Dockerfile (bakes secrets into image history). Do not bind-mount `.env` into a container (creates a second source of truth that drifts from compose's substitution). The only correct path for an `.env` value to reach a process is:

```yaml
services:
  steam-proxy-web:
    environment:
      WEB_PORT: ${WEB_PORT:-3000}
      DB_PASSWORD: ${DB_PASSWORD}
```

### `.env.template` is always present and current

`.env` is gitignored. `.env.template` is committed and lists every variable the stack uses, with comments explaining each and reasonable defaults. By **Inline documentation beats external docs**, the template is the documented surface area of the stack's configuration.

The rule: every variable referenced in the compose file has an entry in `.env.template`. When compose adds a new `${SOMETHING}` substitution, `.env.template` gains a corresponding line. The two files stay in sync.

```bash
# .env.template (committed)

# The compose project namespace, identifying which instance of this product the
# stack belongs to. Use <product>-main for the primary instance, <product>-<branch>
# for parallel feature instances, <product>-prod-blue/green for blue/green deploys.
# Leave as "prod" on single-instance production hosts.
COMPOSE_PROJECT_NAME=prod

# The deployment mode. One of: dev, prod. Defaults to prod for safety.
APP_MODE=prod

# The Dockerfile target to build. One of: dev, runtime. Defaults to runtime for safety.
BUILD_TARGET=runtime

# The host interface to bind published ports to. 127.0.0.1 means localhost-only;
# 0.0.0.0 exposes on every host interface (only set if a reverse proxy is not fronting the stack).
BIND_IP=127.0.0.1

# The host port for the web service.
WEB_HOST_PORT=3000

# Required: the master key for Meilisearch. Generate with: openssl rand -hex 32
MEILI_MASTER_KEY=

# Required: the Postgres user password. Generate with: openssl rand -hex 32
DB_PASSWORD=
```

Add `config.ini.template` (or `config.example.ini`) alongside, with the same approach.

### Defaults in the compose file

Every variable that has a sensible default uses `${VAR:-default}` substitution. Variables without defaults are required — compose errors at startup if they are not set, which is the right behavior for secrets and other values that have no safe fallback.

```yaml
# Has a safe default; works without .env setup.
ports:
  - "${BIND_IP:-127.0.0.1}:${WEB_HOST_PORT:-3000}:3000"

# Required; no default. Compose errors if not set.
environment:
  DB_PASSWORD: ${DB_PASSWORD}
```

By **Production by default**, defaults are always production-safe. A bare `docker compose up` from a fresh checkout produces a hardened stack, not a permissive one.

## Comment style for Dockerfiles, compose, and `.env`

The same comment discipline applies across all Docker-related files (Dockerfile, `docker-compose.yml`, `.env`, `.env.template`, `config.ini`, `config.ini.template`). The medium changes (`#` is the comment marker in all of them) but the voice does not.

### Rules

Comments are sentence fragments, not full sentences. Start with "Used to...", "Used as...", "The...", or an imperative verb ("Mount...", "Bind...", "Wait for..."). Describe the role, not the mechanics — the configuration shows *how*, the comment shows *what for*. Do not narrate obvious config; no `# set the port` above `WEB_PORT=3000`. Comment the non-obvious: workarounds, why a defensive default exists, host-side setup requirements, links to upstream issues or docs. One comment per declaration, one declaration per comment. Voice is impersonal and present-tense: no "we", no "I", no "this will...". `TODO:` and `NOTE:` markers are encouraged for known issues and non-obvious behavior — be honest about rough edges.

Host-side setup commands go inline as comments above the relevant compose line. By **Inline documentation beats external docs**, the format is a comment block directly above the line, with literal commands to run.

### Examples

In a Dockerfile, commenting only what is not obvious:

```dockerfile
# The build args used to align the in-image user with the host's uid/gid.
ARG APP_UID=1000
ARG APP_GID=1000

# Replace the default node user's uid/gid with the host's. In-place mutation
# preserves the home directory, group memberships, and npm cache path.
RUN groupmod -g ${APP_GID} node \
    && usermod -u ${APP_UID} -g ${APP_GID} node

# The healthcheck hits the app's /health endpoint, not just port liveness.
# Distroless has no shell utilities; use Node directly.
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/health', r => process.exit(r.statusCode === 200 ? 0 : 1)).on('error', () => process.exit(1))"
```

In a compose file, with host-side setup documented inline:

```yaml
services:
  steam-proxy-meilisearch:
    image: getmeili/meilisearch:v1.12
    # Volume ownership: Meilisearch runs as uid 1000:1000 inside the image.
    # Before first run:
    #   sudo mkdir -p ${MEILI_DATA_PATH:-./meili-data}
    #   sudo chown -R 1000:1000 ${MEILI_DATA_PATH:-./meili-data}
    volumes:
      - ${MEILI_DATA_PATH:-./meili-data}:/meili_data:Z
```

In `.env.template`, role-describing rather than type-restating:

```bash
# The host interface to bind published ports to. 127.0.0.1 means localhost-only.
BIND_IP=127.0.0.1

# Required: the master key for Meilisearch. Generate with: openssl rand -hex 32
MEILI_MASTER_KEY=

# NOTE: DB_PASSWORD has no default. Compose errors at startup if not set.
DB_PASSWORD=
```

### Bad comments to avoid

Narrating mechanics line by line:

```dockerfile
# Copy package.json
COPY package.json ./
# Run npm ci
RUN npm ci
```

Restating the variable as a comment:

```bash
# WEB_HOST_PORT is the web port
WEB_HOST_PORT=3000
```

Full sentences in first person:

```yaml
# We mount the data volume here. It's where we keep all the user uploads.
volumes:
  - ${DATA_PATH:-./data}:/app/data:Z
```

The good versions are silent on the obvious, vocal on what matters: a one-line role description ("Application data: user uploads, generated reports"), the literal chown command when ownership is non-obvious, the `NOTE:` when a value has no safe default.

### Section banners

For long compose files (three or more services), use banner comments to separate major regions. They make `docker-compose.yml` scannable when it grows past one screen:

```yaml
# ---------------------------------------------------------------------------
# State services
# ---------------------------------------------------------------------------

services:
  steam-proxy-db:
    # ...

# ---------------------------------------------------------------------------
# Application services
# ---------------------------------------------------------------------------

  steam-proxy-web:
    # ...
```

For a short compose file with one or two services, banners are noise; skip them.

## Dev vs. prod mode

Three independent variables together describe the stack's deployment context. They are orthogonal — each does one thing and does not constrain the others. All three default to production-safe values per **Production by default**.

- **`APP_MODE`** — the deployment mode. Defaults to `prod`. Determines hardening defaults across the stack.
- **`BUILD_TARGET`** — which Dockerfile stage gets built. Defaults to `runtime`. Determines whether the image is hardened production or the dev image with shell, watcher, and bind mount.
- **`NODE_ENV`** — what the application sees. Defaults to `production`. Affects framework behavior (Express's error pages, React's dev warnings, npm's install defaults).

Default behavior (no `.env` or production `.env`) sets all three to production values. A dev or staging instance's `.env` opts each one into dev:

```bash
COMPOSE_PROJECT_NAME=steam-proxy-main
APP_MODE=dev
BUILD_TARGET=dev
NODE_ENV=development
```

### Why three variables instead of one

The three vary independently. `APP_MODE=prod` with `BUILD_TARGET=dev` is a useful combination for testing production behavior with the dev image's debugging affordances. `NODE_ENV=production` on a developer machine reproduces a production bug locally. Bundling them under a single `MODE` flag would force changing all three when only one is meant. Keep them separate.

### `COMPOSE_PROJECT_NAME` is an instance namespace, not a mode

Per the principle of the same name, `COMPOSE_PROJECT_NAME` only namespaces resources so multiple instances of the same product can coexist on one host without colliding on container names, image tags, volumes, or networks. The semantics are *which instance of this product*, not *which person owns this checkout* or *which mode is this in*.

Typical instance names:

- `<product>-main` — the primary instance for a single-instance deployment.
- `<product>-feature-<topic>` — a parallel instance running a feature branch alongside the main one.
- `<product>-staging` — a staging instance on the same host as production.
- `<product>-prod-blue`, `<product>-prod-green` — two production instances for blue/green deployment.
- `<product>-pr-<number>` — an ephemeral instance per pull request, spun up by CI for review.

Two instances with the same `COMPOSE_PROJECT_NAME` on the same host collide. Two instances with different names are entirely independent — separate containers, separate image tags, separate volumes, separate networks. The default value `prod` is correct for the common case of a single production instance per host; multi-instance setups (blue/green, staging-alongside-prod, ephemeral per-PR) require explicit naming.

None of this tells the stack whether it is in dev or prod mode — that is `APP_MODE`'s job. A `COMPOSE_PROJECT_NAME=steam-proxy-prod-blue` stack with `APP_MODE=prod` is one of two production instances. A `COMPOSE_PROJECT_NAME=steam-proxy-feature-search` stack with `APP_MODE=dev` is a dev-mode instance running a feature branch.

The compose file never reads `COMPOSE_PROJECT_NAME` to decide behavior. It only uses it as a prefix on resource names.

### `ARG` and `ENV` in the Dockerfile

`ARG` is for things needed only during the build (build numbers, target architectures, `APP_UID`/`APP_GID`). Available in `RUN` steps, not at runtime. Never for secrets.

`ENV` is for runtime configuration with sensible defaults (`NODE_ENV=production` baked into the runtime stage, `PORT=3000`). Available to the running process. Never for secrets.

Neither is the right home for anything sensitive — see **No copies of secrets**.

Set `NODE_ENV=production` in the runtime stage as defense in depth:

```dockerfile
FROM gcr.io/distroless/nodejs20-debian12 AS runtime
ENV NODE_ENV=production
```

Compose's `environment: { NODE_ENV: ${NODE_ENV:-production} }` overrides this at run time, which is correct — the compose value wins, the Dockerfile value is the fallback if compose forgets to set it.

## Base images and the multi-stage Dockerfile

Prefer official Node.js images. Within them:

- **`node:<version>-slim`** is the dev default. Debian-based, small enough, includes a shell and package manager for debugging.
- **`node:<version>-alpine`** is smaller but uses musl libc. Some native modules break under it. Use only when verified.
- **`gcr.io/distroless/nodejs<version>-debian12`** is the production default. No shell, no package manager, no anything but Node. Minimal post-exploitation surface.

Pin to a specific version, never `latest` or a bare major. For high-security contexts, pin to a digest as well: `FROM node:20.18.1-slim@sha256:abc...`. The digest pin protects against tag reassignment.

### The canonical multi-stage Dockerfile

By **Production never copies dev, dev never bakes production**, the dev and runtime targets are different stages of the same file. The shape:

```dockerfile
# syntax=docker/dockerfile:1.7

# Stage 1: builder. Has npm, source, all deps including dev.
FROM node:20.18.1-slim AS builder

ARG APP_UID=1000
ARG APP_GID=1000

# Replace the default node user's uid/gid with the host's. In-place mutation
# preserves the home directory, group memberships, and npm cache path.
RUN groupmod -g ${APP_GID} node \
    && usermod -u ${APP_UID} -g ${APP_GID} node \
    && mkdir -p /app \
    && chown -R node:node /app

WORKDIR /app
USER node

# Order: lockfile first, then source. Changing source does not invalidate
# the dependency-install layer.
COPY --chown=node:node package.json package-lock.json ./
RUN --mount=type=cache,target=/home/node/.npm npm ci

COPY --chown=node:node . .
RUN npm run build

# Prune dev dependencies for the runtime stage.
RUN npm prune --omit=dev

# Stage 2: production runtime. Distroless, pruned node_modules.
FROM gcr.io/distroless/nodejs20-debian12 AS runtime

LABEL org.opencontainers.image.source="https://github.com/org/repo"

ENV NODE_ENV=production

WORKDIR /app

COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./package.json

USER nonroot

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/health', r => process.exit(r.statusCode === 200 ? 0 : 1)).on('error', () => process.exit(1))"

CMD ["dist/server.js"]

# Stage 3: dev. Same toolchain as builder, runs the dev server.
FROM builder AS dev

CMD ["npm", "run", "dev"]
```

The builder runs with full dev dependencies and builds the app. The runtime stage copies the pruned `node_modules` and built `dist/` only — no source, no dev deps, no build toolchain, no shell. The dev stage reuses the builder, so it has everything the builder has for hot-reloading.

Distroless's `CMD` is implicit Node — the image has `ENTRYPOINT ["/nodejs/bin/node"]` baked in, so `CMD ["dist/server.js"]` becomes `node dist/server.js`. Do not write `node` in the CMD line.

Build production with `docker build --target runtime`, dev with `docker build --target dev` (or via compose, which handles it through `BUILD_TARGET`).

### Layer ordering, `npm ci`, `.dockerignore`

`COPY` instructions go from least-frequently-changed to most-frequently-changed. Lockfiles change rarely, source changes constantly. Putting source first invalidates the dependency-install cache on every code change.

Always `npm ci` in a Dockerfile, never `npm install`. `npm ci` reads the lockfile and fails if it disagrees with `package.json`, which is what you want in CI and image builds.

`.dockerignore` is mandatory. Without one, the build context balloons, `.env` and `.git` leak into the image, and `COPY . .` invalidates the cache on any file change. A reasonable starting point:

```
.git
.gitignore
node_modules
npm-debug.log
.env
.env.*
*.log
.DS_Store
coverage
.nyc_output
.vscode
.idea
dist
build
README.md
```

Include the build outputs (`dist`, `build`) — the image rebuilds them, and copying stale local builds into the image is a recurring source of "works on my laptop" bugs.

## Run as non-root, match host uid/gid

By **Assume compromise**, the production container runs as the image's baked-in non-root user. If an attacker gets RCE as `node` (uid 1000), they cannot install packages or modify system files without a separate privilege escalation. If they get RCE as root, the chain is much shorter.

The production stage is one line: `USER nonroot` (distroless) or `USER node` (slim-based). The interesting work is in dev, where the container's uid must match the host's so bind-mounted writes have the right ownership.

### Why both `build.args` and `user:`

Two halves do different jobs. By **Friction over magic**, both halves stay because the failure mode of either alone is silent. Build-time `APP_UID`/`APP_GID` args ensure the image's filesystem (`/app`, npm cache, the user's home) is owned by the host's uid from the moment the image is built. Run-time `user:` is a safety net for cases where the image was built with default args but invoked with different `UID`/`GID` (stale cached layer, pulled image, CI invocation with different env vars).

If only `user:` were set, the container would hit "permission denied" writing to image-owned paths. If only `build.args` were set, a stale image build would silently write files with the wrong uid. Both together means the system self-corrects.

The Dockerfile dev stage shown above takes `APP_UID`/`APP_GID` as build args and re-homes the `node` user in place via `usermod`/`groupmod`. The compose file passes `${UID:-1000}` and `${GID:-1000}` both as build args and as the runtime `user:` directive.

`UID` is exported automatically by bash and zsh; the `:-1000` fallback covers shells that do not and CI environments. `GID` is rarely exported automatically; the 1000 fallback is correct for single-user Linux installs. Developers whose primary gid differs can `export GID=$(id -g)` in their shell.

## Containment: assume compromise

The rules in this section are not about preventing intrusion. They are about limiting what an attacker can do once they are inside. By **Assume compromise**, treat every dependency as potentially compromised, every input as potentially malicious, every runtime tool inside the container as a weapon if it falls into the wrong hands.

The goal: an attacker who gains code execution as the app user finds an environment so stripped-down that the compromise is contained to the application's runtime. No path to the host. No reachable network beyond what the app already needed. No persistence across restart.

### Remove the post-exploitation toolkit

The first thing an attacker does after RCE is reconnaissance and lateral movement. Both depend on tools that should not be in the production image.

Keep these out of the runtime image: shells (`sh`, `bash`, `dash`, `ash`); network tools (`curl`, `wget`, `nc`, `socat`, `ssh`, `scp`); package managers (`apt`, `apk`, `yum`, `npm`); other interpreters (`python`, `perl`, `ruby`); compilers and build tools (`gcc`, `make`); editors and debuggers (`vim`, `nano`, `gdb`, `strace`, `tcpdump`).

The multi-stage build handles most of this. Anything installed in the builder stage does not exist in runtime. The discipline is: do not add convenience tools to the runtime stage just because debugging is easier with them. Put them in the `dev` stage.

Distroless is the strict version of this rule. Going from `node:slim` (still has shell and `apt`) to distroless (Node and almost nothing else) is the single biggest reduction in post-exploitation capability available in a Dockerfile.

### Read-only root filesystem

Run production containers with `--read-only` and tmpfs mounts for whatever needs to be writable:

```bash
docker run \
    --read-only \
    --tmpfs /tmp \
    --tmpfs /run \
    myapp
```

An attacker with code execution then cannot write a persistence mechanism, modify existing files to inject backdoors, or replace the app's own code to survive a restart.

If the app genuinely needs to write somewhere, mount that path explicitly as a tmpfs (`--tmpfs /app/cache`) or a bind mount (`-v ./cache:/app/cache`). The forcing function of read-only-by-default catches accidental writes to the working directory, log files written next to the binary, and other patterns that should have been explicit.

### Drop Linux capabilities

By default, Docker grants containers a permissive set of Linux capabilities. Most Node apps need none of them.

```bash
docker run \
    --cap-drop=ALL \
    --security-opt=no-new-privileges \
    myapp
```

The one common exception is binding to ports below 1024 (`NET_BIND_SERVICE`). The right answer is usually not to — bind to a high port and let a reverse proxy handle privileged binding. `no-new-privileges` prevents the process from gaining new privileges via setuid binaries even if one somehow ended up in the image.

### Network egress and ingress

By **Production by default**, network exposure is opt-in. The compose file ships with safe defaults; `.env` is where exposure decisions live.

**Egress.** Docker's segmentation tools are custom networks and `network_mode: none`. Containers on a named bridge network can reach each other and the host's default gateway but are segmented from other networks on the same host. Containers with `network_mode: none` cannot reach the network at all — useful for batch jobs and file processors that only read and write volumes.

```yaml
services:
  steam-proxy-web:
    networks:
      - internal
  
  steam-proxy-batch:
    network_mode: none

networks:
  internal:
    driver: bridge
```

Hostname-level egress allow-lists are not a Docker feature. They are handled by infrastructure outside the scope of this document.

**Ingress.** The default `ports: ["3000:3000"]` binds to `0.0.0.0:3000` — every host interface, reachable from anywhere that can reach the host. On a laptop on public Wi-Fi, that is everyone on the same network. On a server, the public internet.

Always bind to a specific IP and port both configurable via `.env`:

```yaml
ports:
  - "${BIND_IP:-127.0.0.1}:${WEB_HOST_PORT:-3000}:3000"
```

`BIND_IP=127.0.0.1` is the default. Set `BIND_IP=0.0.0.0` in `.env` only when public exposure is genuinely intended. For most production deployments the right shape is: app container on `127.0.0.1:<port>`, a reverse proxy on the host bound to `0.0.0.0:80` and `0.0.0.0:443`, proxying to the app. The reverse proxy is the only thing the public internet can reach.

Ensure that each service is using a different IP binding env variable. For instance a webapp should not use the same IP env variable as the database.

### Host service access: dev-only

`extra_hosts: ["host.docker.internal:host-gateway"]` gives the container a DNS name resolving to the host. In dev, this is sometimes genuinely needed (the app calls a service running directly on the host). In production it is a major hardening violation — a compromised container can reach every service the host runs (SSH, metrics, the host firewall's admin port).

Never add `extra_hosts` to production services. Add to specific dev services only when actually needed, with a comment explaining why:

```yaml
services:
  steam-proxy-web:
    # extra_hosts only because dev calls a local debugger on the host.
    # Must not appear in production deployment.
    extra_hosts:
      - "host.docker.internal:host-gateway"
      - "host.containers.internal:host-gateway"
```

The two host names cover Docker and Podman conventions respectively, so the compose file works under either runtime.

For production, prefer reaching what would otherwise be host services via service-to-service URLs on a private network. Run them in their own containers, on the compose network, addressed by service name.

### Anti-patterns

The following compose patterns directly violate the principles in this guide and must not appear in production:

- **`privileged: true`** gives the container root-equivalent access to the host, including device access and capability passthrough. Almost never needed. If a container needs a specific capability, use `cap_add` to grant only that one.
- **`network_mode: host`** removes container network isolation entirely. The container shares the host's network stack and binds to host interfaces directly. Use bridge networks and explicit port publishing instead.
- **Disabling `cap_drop: ALL`** to give a container "all the capabilities it might need." The default-drop-all is what makes additive `cap_add` meaningful.
- **Hardcoded secrets in compose** (a literal `MEILI_MASTER_KEY=abc123` in `environment:`). Use `${MEILI_MASTER_KEY}` with no default; compose errors at startup if missing, which is correct.
- **Boot-time `npm install`** in a `command:` block. See the cron section for the full argument — the short version is that it breaks first-run population, makes the image incomplete, and adds a network round-trip to every boot.
- **`crond` running inside a container with a heredoc-generated crontab.** See the cron section.

### The threat model in one paragraph

Assume one of your dependencies is malicious or compromised. Assume it will execute code as the app's user inside the container. Now ask: what tools does it have? What can it write? Where can it connect? What secrets can it read? What can it leave behind that survives a restart? Each "yes" on those questions is a hardening opportunity. The goal is a container where the attacker has Node, the app's code, and nothing else — no shell, no network reach beyond approved destinations, no writable filesystem, no extra credentials, no persistence mechanism. Stuck inside Node, inside one process, with limited time before the next deploy wipes them out.

## The canonical service shape

Every application service in `docker-compose.yml` follows the same shape. State services follow a related but distinct shape (covered later). The canonical shape is defined once here; later sections refer to it by name instead of restating it.

```yaml
services:
  <service-name>:
    build:
      context: .
      target: ${BUILD_TARGET:-runtime}
      args:
        APP_UID: "${UID:-1000}"
        APP_GID: "${GID:-1000}"
    image: ${COMPOSE_PROJECT_NAME:-prod}-<project>-<service-role>:local
    container_name: ${COMPOSE_PROJECT_NAME:-prod}-<project>-<service-role>
    working_dir: /app
    user: "${UID:-1000}:${GID:-1000}"
    userns_mode: "keep-id"
    init: true
    restart: unless-stopped
    environment:
      TZ: ${TZ:-Europe/London}
      NODE_ENV: ${NODE_ENV:-production}
    ports:
      - "${BIND_IP:-127.0.0.1}:${<ROLE>_HOST_PORT:-3000}:3000"
    volumes:
      - ./:/app
      - <project>_<service-role>_node_modules:/app/node_modules
      - ${CONFIG_PATH:-./config.ini}:/app/config.ini:ro
      - ${DATA_PATH:-./data}:/app/data:Z
      - ${LOGS_PATH:-./logs}:/app/logs:Z

volumes:
  <project>_<service-role>_node_modules:
```

### What each piece does

- `build.args.APP_UID`/`APP_GID` — by **Friction over magic**, build-time uid/gid match the host's so the image's filesystem is owned correctly from build.
- `image:` — locally-built tag, prefixed by `COMPOSE_PROJECT_NAME`. The `:local` suffix distinguishes locally-built images from anything pulled from a registry.
- `container_name:` — runtime name shown in `docker ps`, prefixed by instance namespace to prevent collisions per **`COMPOSE_PROJECT_NAME` is an instance namespace**.
- `working_dir: /app` — explicit working directory, matches what the Dockerfile sets up.
- `user:` — runtime uid/gid as belt-and-braces alongside the build args.
- `userns_mode: "keep-id"` — Podman-specific; required for the Podman dev workflow, ignored by Docker (with a harmless warning). Kept in the canonical shape so a single compose file works under both runtimes.
- `init: true` — wraps the container in an init process for proper signal forwarding (graceful shutdown, zombie reaping).
- `restart: unless-stopped` — auto-restart on crash so transient failures heal themselves.
- `environment.TZ` — IANA timezone for cron schedules, log timestamps, date formatting. Without it, container defaults to UTC.
- `environment.NODE_ENV` — runtime mode, defaults to `production` per **Production by default**.
- `ports:` — published port with configurable bind IP and host port. `BIND_IP` defaults to `127.0.0.1` (localhost only); host port follows the role-prefix naming convention (`WEB_HOST_PORT`, `CRON_HOST_PORT` if applicable). Services that do not accept inbound traffic omit `ports:` entirely.
- `volumes:` — source bind mount in dev, named volume for `node_modules`, read-only bind mount for `config.ini`, configurable bind mounts for data and logs.

### Concrete instantiation

For a project called `steam-proxy` with a web service and a cron service:

```yaml
services:
  steam-proxy-web:
    build:
      context: .
      target: ${BUILD_TARGET:-runtime}
      args:
        APP_UID: "${UID:-1000}"
        APP_GID: "${GID:-1000}"
    image: ${COMPOSE_PROJECT_NAME:-prod}-steam-proxy-web:local
    container_name: ${COMPOSE_PROJECT_NAME:-prod}-steam-proxy-web
    working_dir: /app
    user: "${UID:-1000}:${GID:-1000}"
    userns_mode: "keep-id"
    init: true
    restart: unless-stopped
    environment:
      TZ: ${TZ:-Europe/London}
      NODE_ENV: ${NODE_ENV:-production}
    ports:
      - "${BIND_IP:-127.0.0.1}:${WEB_HOST_PORT:-3000}:3000"
    volumes:
      - ./:/app
      - steam_proxy_web_node_modules:/app/node_modules
      - ${CONFIG_PATH:-./config.ini}:/app/config.ini:ro
      - ${DATA_PATH:-./data}:/app/data:Z
      - ${LOGS_PATH:-./logs}:/app/logs:Z

  steam-proxy-cron:
    build:
      context: .
      target: ${BUILD_TARGET:-runtime}
      args:
        APP_UID: "${UID:-1000}"
        APP_GID: "${GID:-1000}"
    image: ${COMPOSE_PROJECT_NAME:-prod}-steam-proxy-cron:local
    container_name: ${COMPOSE_PROJECT_NAME:-prod}-steam-proxy-cron
    working_dir: /app
    user: "${UID:-1000}:${GID:-1000}"
    userns_mode: "keep-id"
    init: true
    restart: unless-stopped
    environment:
      TZ: ${TZ:-Europe/London}
      NODE_ENV: ${NODE_ENV:-production}
    volumes:
      - ./:/app
      - steam_proxy_cron_node_modules:/app/node_modules
      - ${CONFIG_PATH:-./config.ini}:/app/config.ini:ro
      - ${DATA_PATH:-./data}:/app/data:Z
      - ${LOGS_PATH:-./logs}:/app/logs:Z
    command: ["node", "scripts/scheduler.js"]

volumes:
  steam_proxy_web_node_modules:
  steam_proxy_cron_node_modules:
```

The cron service is identical to the web service except: no `ports:` (it does not accept inbound traffic), and an explicit `command:` overriding the default `CMD`. Both share image, build args, user mapping, volume mounts, and environment.

### Naming shape

`<instance>-<product>-<service-role>` reads cleanly in `docker ps`:

```
steam-proxy-main-steam-proxy-web
steam-proxy-main-steam-proxy-cron
steam-proxy-feature-search-steam-proxy-web
steam-proxy-prod-blue-steam-proxy-web
steam-proxy-prod-green-steam-proxy-web
prod-billing-api-web
```

The instance name (`steam-proxy-main`, `steam-proxy-prod-blue`) often repeats the product name as its own prefix, which looks redundant until you have multiple products on the same host. Then `steam-proxy-prod-blue-steam-proxy-web` next to `billing-api-prod-blue-billing-api-web` reads exactly as it should — instance grouping first, service identification second.

For single-instance deployments, the default `COMPOSE_PROJECT_NAME=prod` produces simpler names: `prod-steam-proxy-web`, `prod-steam-proxy-cron`. The instance-named form is for the multi-instance cases (blue/green, feature branches, ephemeral environments).

### What not to prefix

By **`COMPOSE_PROJECT_NAME` is an instance namespace**, compose namespaces some things automatically: service names (used for inter-container DNS), top-level volume names, top-level network names. Do not write `${COMPOSE_PROJECT_NAME}-` inside `depends_on:` references, inter-service URLs, or top-level volume declarations. Use bare service names; compose's network resolves them correctly.

## Service dependencies and failure isolation

A service declares `depends_on` only for what it cannot start without. Not "would be nice to have running"; "would crash on startup without."

Over-declaring `depends_on` creates two problems. Startup coupling: a service waits for things it does not actually need, slowing boot and increasing the chance of startup-order failures. Shutdown coupling: `docker compose stop <service>` brings down everything that depends on it. Stopping the database to swap it out should not take down the search indexer.

### Use conditions, not the short form

The short form `depends_on: [foo]` waits for `foo` to start but does not wait for `foo` to be ready and does not restart dependents if `foo` crashes. Use the long form with conditions:

```yaml
services:
  steam-proxy-web:
    depends_on:
      steam-proxy-db:
        condition: service_healthy
      steam-proxy-meilisearch:
        condition: service_started
```

Three conditions: `service_started` waits for the dependency's container to be running (cheap, no healthcheck needed, weakest guarantee). `service_healthy` waits for the dependency's `HEALTHCHECK` to pass (stronger, requires the dependency to define one — use for services dependents will connect to immediately, especially databases). `service_completed_successfully` waits for the dependency to run and exit 0 (use for one-shot containers like migrations).

### Optional dependencies belong in application code

A web service that calls a search index should not list the search in `depends_on` if the web can serve traffic without it (cached pages, database fallback). Use timeouts and retries in application code when calling the search. Log degraded behavior so it is visible in monitoring.

`depends_on` describes "cannot boot without this." Application-level resilience describes "can keep running even if this becomes unavailable mid-flight." Different problems, different solutions.

By **Compose is deployment, code is application**, the boot-dependency boundary belongs in compose; the runtime-resilience boundary belongs in code.

### `restart: unless-stopped` handles transient failures

The canonical shape includes `restart: unless-stopped`. With it: a dependency crashes and restarts; dependents that crash because they lost their dependency also restart; after both restart, the dependency-condition recheck gates the order. Transient failures heal without intervention.

## Volumes

By **Data and artifacts split**, every volume is one of two things: persistent data (bind mount) or derived artifact (named volume). The test is "could I regenerate this from source?" — yes means named volume, no means bind mount.

### Persistent data: always bind-mounted

Anything the application persists — uploads, generated files, databases, search indexes, log files, caches that should survive a restart — gets a host path for storage and a container path for where the app expects to read and write. The host path is configurable; the container path is fixed by the app's expectations.

```yaml
volumes:
  - ${DATA_PATH:-./data}:/app/data:Z
  - ${LOGS_PATH:-./logs}:/app/logs:Z
  - ${CACHE_PATH:-./cache}:/app/cache:Z
```

Defaults resolve relative to the directory containing `docker-compose.yml`, so a bare `docker compose up` from a fresh checkout creates `./data`, `./logs`, `./cache` next to the source tree.

In `.env`, production overrides to real system paths:

```bash
DATA_PATH=/var/lib/steam-proxy/data
LOGS_PATH=/var/log/steam-proxy
CACHE_PATH=/var/cache/steam-proxy
```

Gitignore the defaults at the repo root:

```
/data/
/logs/
/cache/
```

Reasons for bind-mounting persistent data per **Data and artifacts split**: bind mounts put data on a known host path that can be backed up by copying the directory; `docker compose down -v` does not touch bind mounts (named volumes can be wiped accidentally); data files are inspectable from the host without going through Docker; production deployment paths are predictable.

### `:Z` for SELinux relabeling

On SELinux-enforcing hosts (Fedora, RHEL, CentOS Stream, Rocky, Alma), bind-mounted directories need their SELinux context relabeled. Without it, expect cryptic "permission denied" errors that look like file permission problems but are not.

Always append `:Z` (uppercase, private context) to bind mounts:

```yaml
volumes:
  - ${DATA_PATH:-./data}:/app/data:Z
```

`:z` (lowercase, shared context) is for the rare case of multiple containers genuinely needing write access to the same directory; usually a sign the architecture should be revisited. On non-SELinux hosts the suffix is harmless and ignored, so add it by default to keep the compose file portable.

### Read-only mounts where possible

If the container only reads from a bind mount (config files, certificates, static assets), mount it read-only with `:ro`:

```yaml
volumes:
  - ${CONFIG_PATH:-./config.ini}:/app/config.ini:ro
  - ${CERTS_PATH:-./certs}:/etc/ssl/certs:ro
```

By **Assume compromise**, read-only is the right default for anything the container does not need to write. A compromised container with code execution cannot modify source-of-truth config or rotate certificates if the mount is read-only.

### Derived artifacts: always named volumes

`node_modules` and similar derived artifacts are named volumes scoped per-service. By **Data and artifacts split**, the test is whether the volume can be regenerated from source. `node_modules` regenerates from `package.json` and `package-lock.json`, so it is a named volume. The same applies to build caches (`.turbo`, `.next`, `.vite`).

Volume names follow the same `<project>_<service-role>_<artifact>` shape as image tags and container names, without the `${COMPOSE_PROJECT_NAME}-` prefix (compose adds it):

```yaml
services:
  steam-proxy-web:
    volumes:
      - steam_proxy_web_node_modules:/app/node_modules

volumes:
  steam_proxy_web_node_modules:
```

Each service gets its own `node_modules` volume because each service may have a different `node_modules` shape (the web service might run with pruned production deps; a build service might have the full dev install).

### How `node_modules` gets populated

`node_modules` is baked into the image during build. The named volume is a runtime mount that happens to be writable. Both are true simultaneously, and the interaction is the key to making this work:

When the container starts and the named volume mounts over `/app/node_modules`, Docker checks if the volume is empty. If empty (first run, or after `docker compose down -v`), Docker copies the image's `/app/node_modules` into the volume — the container starts with a populated `node_modules` matching the image. If the volume already has contents, Docker uses what is there; the volume diverges from the image as `npm install` runs inside the container.

The image must always be self-sufficient. An image that requires a non-empty named volume to function is broken. By **Production never copies dev, dev never bakes production**, the image is the source of truth; the named volume is for *persistence of runtime mutations*, not for *first source of truth*.

Workflow: add a dependency with `docker compose exec <service> npm install <package>`, which writes to the named volume and to `package.json`/`package-lock.json` on the host via the source bind mount. Commit the lockfile. After pulling new code with new dependencies, either `docker compose exec <service> npm install` to sync the volume or `docker compose down -v && docker compose up` to fully reseed from a fresh image build.

Boot-time `npm install` in compose `command:` breaks this model (the image becomes incomplete; the volume becomes the only source of truth). See the cron section anti-pattern for the full argument.

### Inline documentation of host-side setup

By **Inline documentation beats external docs**, anything the compose file expects to exist on the host before `docker compose up` works is documented inline as a comment above the relevant line.

Common cases: volume ownership (bind-mounted directories that need to be owned by a specific uid for the container to write to them); required host directories (directories that must exist with specific permissions before the bind mount works); required host config files (config files the container reads via bind mount); required environment variables with no default (secrets that fail at compose startup if missing).

The format is a comment block directly above the line, with literal commands:

```yaml
services:
  steam-proxy-db:
    # Volume ownership: Postgres runs as uid 999:999 inside the image.
    # Before first run:
    #   sudo mkdir -p ${DB_DATA_PATH:-./db-data}
    #   sudo chown -R 999:999 ${DB_DATA_PATH:-./db-data}
    volumes:
      - ${DB_DATA_PATH:-./db-data}:/var/lib/postgresql/data:Z

    # Required env vars (no defaults; compose errors if missing):
    #   DB_PASSWORD - Postgres user password, generate with: openssl rand -hex 32
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
```

A fresh checkout plus `docker compose up` should fail with a clear message if a setup step is missing, and the message plus the inline comment together should give the reader everything they need to fix it. No separate `SETUP.md` to consult.

## State services

Database, search, cache, and message-queue containers follow a related but distinct shape. They use official upstream images, persist data via configurable bind mounts (per **Data and artifacts split**), and expose internal ports for inter-service traffic and host ports for inspection only.

### The state service shape

```yaml
services:
  steam-proxy-meilisearch:
    image: getmeili/meilisearch:v1.12
    container_name: ${COMPOSE_PROJECT_NAME:-prod}-steam-proxy-meilisearch
    restart: unless-stopped
    ports:
      - "${BIND_IP:-127.0.0.1}:${MEILI_HOST_PORT:-7700}:7700"
    # Volume ownership: Meilisearch runs as uid 1000:1000 inside the image.
    # Before first run:
    #   sudo mkdir -p ${MEILI_DATA_PATH:-./meili-data}
    #   sudo chown -R 1000:1000 ${MEILI_DATA_PATH:-./meili-data}
    volumes:
      - ${MEILI_DATA_PATH:-./meili-data}:/meili_data:Z
    environment:
      TZ: ${TZ:-Europe/London}
      MEILI_ENV: ${MEILI_ENV:-production}
      MEILI_MASTER_KEY: ${MEILI_MASTER_KEY}
      MEILI_NO_ANALYTICS: "true"
```

Differences from the canonical app service shape: upstream image pinned to a specific version (never `latest`); no `build:` block; no `user:` or `userns_mode:` overrides (the image's baked-in user is what runs); data volume always bind-mounted with documented chown; secrets have no default (`MEILI_MASTER_KEY: ${MEILI_MASTER_KEY}`).

### Volume ownership: the upstream uid problem

Many official state-service images bake in a specific uid for the data directory:

| Image | Internal uid:gid |
| --- | --- |
| Meilisearch | 1000:1000 |
| Postgres | 999:999 |
| Redis | 999:999 |
| MongoDB | 999:999 |
| Elasticsearch | 1000:0 |
| RabbitMQ | 999:999 |

You cannot change this with a build arg because the image is not being built. `user:` at runtime does not help because the image's internal data directory is owned by the baked-in uid; running as a different uid means writes fail.

Per **Data and artifacts split** and **Friction over magic**, persistent data is always bind-mounted. The chown is documented inline. No wrapper images, no init containers that re-chown at startup — those add complexity to paper over the question of "who owns this data?" The friction is the point.

The inline comment names the uid:gid and the literal commands so first-time setup is self-documenting:

```yaml
# Volume ownership: Postgres runs as uid 999:999 inside the image.
# Before first run:
#   sudo mkdir -p ${DB_DATA_PATH:-./db-data}
#   sudo chown -R 999:999 ${DB_DATA_PATH:-./db-data}
volumes:
  - ${DB_DATA_PATH:-./db-data}:/var/lib/postgresql/data:Z
```

### Inter-service URLs

App services reach state services by service name on the compose network. The internal port (right side of `ports:`) is what the network sees, regardless of what host port is published.

```js
// In the app
const meiliClient = new MeiliSearch({
    host: process.env.MEILI_URL ?? 'http://steam-proxy-meilisearch:7700',
    apiKey: process.env.MEILI_MASTER_KEY
});
```

`MEILI_URL` lets the app run outside Docker (pointing at a local Meilisearch on `localhost:7700`) without code changes. The host-side published port is for inspection from the host (a `curl`, a GUI client) — never for service-to-service traffic.

### Network isolation

State services should only be reachable by the app services that need them. Compose's default puts everything on one network, which is fine for small stacks. For more isolation:

```yaml
services:
  steam-proxy-web:
    networks: [frontend, backend]

  steam-proxy-meilisearch:
    networks: [backend]

  steam-proxy-public-thing:
    networks: [frontend]

networks:
  frontend:
  backend:
```

The web service is on both networks; it can reach Meilisearch on `backend` and respond to public traffic on `frontend`. A different public-facing service has no business touching Meilisearch and is on `frontend` only — physically cannot reach the search backend. By **Assume compromise**, this limits what a compromised public-facing service can talk to.

### Admin and inspection sidecars

For tooling that inspects a state service (SQL studios, Redis GUIs, queue dashboards, mail catchers), use a separate compose file rather than a service in the main file:

```
docker-compose.yml          # core services, always started
docker-compose.tools.yml    # sidecars, opt-in
```

To start with sidecars, layer the files:

```bash
docker compose -f docker-compose.yml -f docker-compose.tools.yml up
```

Or set the merged file list in `.env`:

```bash
COMPOSE_FILE=docker-compose.yml:docker-compose.tools.yml
```

By **Production by default**, production `.env` omits the tools file, so a bare `docker compose up` on the production host gets only core services. The file split is a stronger boundary than a profile flag: `docker-compose.yml` alone is the production stack, and reviewing that file alone tells you everything that runs in prod.

For occasional production debugging, the tools file can be layered in temporarily and torn down when done:

```bash
docker compose -f docker-compose.yml -f docker-compose.tools.yml up -d steam-proxy-studio
# ... debug ...
docker compose stop steam-proxy-studio
docker compose rm -f steam-proxy-studio
```

The sidecar binds to `127.0.0.1` per the canonical shape, so only someone with SSH access to the host can reach it. For browser access, SSH-tunnel rather than exposing to the network.

## Scheduled jobs

Scheduled jobs run in a dedicated container, separate from the web service. By **Compose is deployment, code is application**, the cron concern is its own deployment artifact with its own lifecycle, logs, and failure surface.

The cron container runs a Node scheduler — a long-lived Node process using a cron library (`node-cron`, `croner`) that schedules and dispatches jobs. It does **not** run `crond`. It does **not** install dependencies at boot. It does **not** write its crontab at runtime via a heredoc.

### Why not `crond` in a container

`crond` is a shell-driven scheduler. Making it work inside a hardened container means adding a shell, coreutils, `crond`, `flock`, `tee` — every one of which is a tool elsewhere in this guide tries to remove. By **Assume compromise**, more tools means more attacker capability.

A Node scheduler needs none of those. The cron container uses the same hardened runtime base image as the web container, runs as the same non-root user, has the same read-only filesystem (where applicable), exposes the same minimal surface.

### The scheduler

```js
// scripts/scheduler.js
import cron from 'node-cron';
import { spawn } from 'node:child_process';

const running = new Set();

/** Used to spawn a worker script as a child Node process, inheriting stdio. */
function spawnScript(script, args = [], extraEnv = {}) {
    return new Promise((resolve, reject) => {
        const child = spawn('node', [script, ...args], {
            stdio: 'inherit',
            env: { ...process.env, ...extraEnv }
        });

        child.on('exit', (code) => {
            if (code === 0) {
                resolve();
            } else {
                reject(new Error(`${script} exited with code ${code}`));
            }
        });
    });
}

/** Used to guard a job against overlapping runs. */
function runOnce(jobName, fn) {
    if (running.has(jobName)) {
        console.warn(`${jobName} still running, skipping this tick`);
        return;
    }

    running.add(jobName);

    Promise.resolve(fn())
        .catch((error) => console.error(`${jobName} failed:`, error.message))
        .finally(() => running.delete(jobName));
}

cron.schedule('0 0 * * *', () => {
    runOnce('refresh', async () => {
        await spawnScript('scripts/download-steamdb.js');
        await spawnScript('scripts/fetch-steam-top-games.js');
    });
});

cron.schedule('0 1 * * *', () => {
    runOnce('descriptions', async () => {
        await spawnScript('scripts/generate-descriptions.js', ['british english', 'gb']);
        await spawnScript('scripts/generate-descriptions.js', ['romanian', 'ro']);
    });
});

console.log('scheduler started');
```

The cron container follows the canonical service shape with `command: ["node", "scripts/scheduler.js"]` and no `ports:`. Scheduling logic lives in version-controlled Node code that gets code review, tests, and the same scrutiny as the rest of the codebase. Environment variables read with `process.env.X ?? 'default'`, not interpolated by the shell inside a YAML heredoc.

### Locking and logging

`runOnce` is the lock that replaces `flock`. The lock lives in the scheduler's process memory; if the scheduler dies, the lock dies with it (which is correct — there is nothing to lock against once the scheduler is gone). No writable `/tmp` for lock files. No `flock` binary needed in the image. The lock represents *current scheduler state*, not historical lock-file artifacts.

If a specific job genuinely needs cross-restart locking (it touches external state that cannot tolerate concurrent access), the right solution is a database row, not a `/tmp` file. SQLite advisory locks, Postgres `pg_advisory_lock`, Redis `SETNX` — all do the job correctly across processes.

The scheduler writes to `console.log`/`console.error`. Child scripts inherit stdio and write to the same. Docker captures it via `docker logs`. The host ships it wherever logs should go. By **Assume compromise** and the read-only-filesystem rule, do not write logs to files inside the container. If logs in a host file are wanted, configure the Docker logging driver:

```yaml
services:
  steam-proxy-cron:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

### Anti-pattern: `crond` inside a container with a generated crontab

The pattern that must not appear:

```yaml
# Don't do this.
services:
  cron:
    command:
      - sh
      - -c
      - |
        npm install --omit=dev &&
        mkdir -p /tmp/crontabs &&
        cat > /tmp/crontabs/app <<EOF &&
        0 0 * * * flock -n /tmp/cron.lock sh -c 'cd /app && node x.js && node y.js 2>&1 | tee -a /var/log/...'
        EOF
        crond -f -L /dev/stdout -c /tmp/crontabs
```

Every line of this fights the rest of the guide:

- Requires a shell (`sh`, `cat`, `tee`, `flock`, `mkdir`, `crond`) — by **Assume compromise** the runtime image should not have one.
- Installs dependencies at boot (`npm install`) — by **Production never copies dev, dev never bakes production** the image should be self-contained.
- Writes the crontab at runtime via a heredoc — schedule logic should live in version-controlled files, not a YAML string.
- Pipes to `tee` to write log files inside the container — fights the read-only filesystem rule.
- Uses `flock` files in `/tmp` — requires coreutils and a writable `/tmp`.

Every one of these is a hardening rule from elsewhere in the guide, broken locally to make this one pattern work. The Node-scheduler approach needs none of them.

## Local development workflow

The dev stage of the Dockerfile and the dev-friendly bits of `docker-compose.yml` (bind-mounted source, named-volume `node_modules`, host uid matching) give a container that behaves like a local environment without polluting the host.

### Auto-reload on source changes

If the framework provides a dev server with watching, use that. Most modern Node frameworks ship dev servers with hot module replacement and route invalidation already wired up; their watchers understand the framework's module graph and reload only what is needed. Examples: `next dev`, `vite dev`, `nest start --watch`, `remix dev`, `astro dev`.

Do not wrap a framework's dev server in `node --watch` or `nodemon` — double watchers cause double restarts and fight the framework's incremental reload.

```json
{
  "scripts": {
    "dev": "vite dev",
    "build": "vite build",
    "start": "node dist/server.js"
  }
}
```

For plain Node apps without a framework dev server, use `node --watch` (built into Node 18.11+) or `tsx watch` (for TypeScript). Do not add `nodemon` to new projects — `node --watch` does what `nodemon` did, natively, without a devDependency.

The compose service runs whatever the `dev` script is:

```yaml
services:
  steam-proxy-web:
    command: ["npm", "run", "dev"]
    volumes:
      - ./:/app
      - steam_proxy_web_node_modules:/app/node_modules
```

Edit a file on the host → bind mount makes it visible inside the container → the watcher sees the change → process reloads → next request hits the new code.

If watching does not pick up changes: polling may be needed on some filesystems (set `NODE_OPTIONS=--watch=polling`); the watcher may not see files outside the watched directory (top-level config changes may need an explicit watch flag or a container restart); some editors use "safe write" rename-over-original semantics that confuse watchers.

### Installing new dependencies

The named volume for `node_modules` means the container owns its own dependencies, separate from the host's. The host's `node_modules` (if any) is irrelevant — it is shadowed by the volume.

To add a dependency, run `npm install` inside the running container:

```bash
docker compose exec steam-proxy-web npm install <package>
```

The new dependency lands in the named volume. `package.json` and `package-lock.json` are updated on the host via the source bind mount. Commit the lockfile.

Do not `npm install` on the host expecting it to affect the container — host and container may have different OS or libc, and copying `node_modules` across causes ABI problems the named volume exists to prevent.

When the named volume's state diverges from the image and a clean rebuild is wanted:

```bash
docker compose down -v
docker compose up
```

`-v` removes named volumes, so the next `up` reseeds `node_modules` from the freshly-built image.

### Config changes

Changes to `.env` are picked up by compose on the next container start, not on the next file change — compose reads `.env` once when it builds its in-memory model of services. To apply:

```bash
docker compose up -d
```

`docker compose up` is idempotent for unchanged services; only services whose environment actually changed get recreated.

Changes to `config.ini` are picked up at the next container start as well, since the file is bind-mounted read-only. For apps that watch their config file at runtime (hot reload of feature flags), the bind mount makes that work automatically — the container sees the host's changes immediately. For apps that read config only at startup, restart the relevant container: `docker compose restart steam-proxy-web`.

### Daily workflow commands

```bash
# Start the stack
docker compose up -d

# Watch logs
docker compose logs -f

# Edit code on the host; auto-reload handles it

# Add a dependency
docker compose exec steam-proxy-web npm install some-package
git add package.json package-lock.json
git commit -m "..."

# Apply a .env change
docker compose up -d

# Get a shell inside a dev container (only works for dev stage)
docker compose exec steam-proxy-web sh

# End of day
docker compose down
```

For multiple services, `docker compose logs -f` follows all of them interleaved. With `pino`/`pino-pretty` for structured logging, dev pipes through the pretty-printer:

```json
{
  "scripts": {
    "dev": "node --watch src/index.js | pino-pretty",
    "start": "node dist/index.js"
  }
}
```

Production logs stay as raw JSON for the host's log shipper.

### Dev containers see the working tree

By **Assume compromise**, a bind-mounted source tree exposes the working directory to the container. A compromised dev container can read `.env` files in the tree, `.git`, editor swap files, anything else.

Mitigations: do not keep real secrets in `.env` files committed near the source — committed `.env.template` files hold development-safe defaults; production secrets live outside the repo (in a secrets manager, in a host-level `.env.local` outside the working tree). `node_modules` is always a named volume, in dev and prod — a compromised package can read its own files but does not get bind-mount semantics into the rest of the host filesystem. For trying out sketchy dependencies, use a throwaway container with no bind mount.

A compromised dev environment is a real risk, but the threat model is "this might leak my dev credentials" rather than "this might compromise production." Keep the two environments fully separated — different credentials, different network segments, different `.env` contents.

### When to rebuild vs. restart

| Change | Action |
| --- | --- |
| Source code | Nothing — watch mode auto-reloads |
| `package.json` (new dep) | `docker compose exec ... npm install` |
| `.env` | `docker compose up -d` |
| `config.ini` (runtime watcher) | Nothing — bind mount makes changes visible |
| `config.ini` (startup-only read) | `docker compose restart <service>` |
| `Dockerfile` | `docker compose build && docker compose up -d` |
| `docker-compose.yml` | `docker compose up -d` (recreates affected services) |
| Node version, OS packages | `docker compose build --no-cache && docker compose up -d` |

### What dev does that prod doesn't

| Aspect | Dev | Prod |
| --- | --- | --- |
| Base image | `node:20-slim` (builder + dev stage) | `gcr.io/distroless/nodejs20-debian12` |
| Shell available | Yes | No |
| Package manager | Yes (`npm`) | No |
| Source bind-mounted | Yes (`./:/app`) | No (baked into image) |
| `node_modules` source | Named volume, seeded from image | Named volume, seeded from image (pruned) |
| Process | `npm run dev` (watcher) | `node dist/server.js` |
| `NODE_ENV` | `development` | `production` |
| `extra_hosts` | Sometimes (dev convenience) | No |
| File writes during runtime | Wherever, fine | Tmpfs and volumes only |

Both stages share: the multi-stage Dockerfile, uid/gid matching, `init: true`, named project prefix, bind-mounted persistent volumes, the `restart: unless-stopped` policy.

## First-time setup

Commands to bring up a fresh instance from a checkout:

```bash
cp .env.template .env
cp config.ini.template config.ini
# Then edit .env to set COMPOSE_PROJECT_NAME (typically <product>-main for the
# default instance, or <product>-<branch> for parallel instances) and any
# required secrets. Do any host-side chown steps named in docker-compose.yml's
# inline comments.
docker compose up
```

By **Inline documentation beats external docs**, the compose file's inline comments name the chown commands for any state services with bind-mounted data. No separate `SETUP.md` to consult — the file is self-documenting.

On a production server: copy `.env.template` to `.env`, fill in production values (real secrets, real host paths, `APP_MODE=prod`, `BUILD_TARGET=runtime`), do the inline-documented chown steps, then `docker compose up -d`.

If `.env` is missing entirely, compose errors on variables without defaults (most secrets). Variables with defaults still work, producing a safe production-shaped stack — but missing whatever the secrets gate. By **Production by default**, that is the right behavior: silently running with a missing secret is worse than failing loudly.

## Runtime split: Podman in dev, Docker in prod

The compose file is written to run under either Podman (typically local dev) or Docker Engine (production). Most of compose's surface is identical between the two. The one visible difference in the canonical shape is `userns_mode: "keep-id"`, which Podman requires and Docker tolerates with a warning.

Features that work identically under both: `build.args`, `image:`, `container_name:`, `volumes:`, `command:`, `init:`, `user:`, `working_dir:`, named volumes and networks, environment files, service dependencies, the `${VAR:-default}` substitution syntax.

Differences worth knowing:

- **Image pulls.** Podman defaults to fully-qualified registry names. Writing `image: somecustom:tag` without `docker.io/library/` prefixing works under Docker but not Podman.
- **Rootless mode.** Podman defaults to rootless, Docker does not. The uid-matching mechanics work under both, but rootless Podman has additional subuid/subgid quirks; "Operation not permitted" errors point to `/etc/subuid` and `/etc/subgid` checks.
- **`docker compose` vs `podman-compose`.** `podman-compose` is a separate project and not 100% compatible. Newer Podman versions ship a native `docker compose` shim that is closer to compatible.

Priority order: production is the constraint, dev can adapt. If a feature is runtime-specific, prefer the version that works under Docker.

## Timezone

By default, most Linux container images run in UTC. Without an explicit `TZ`: cron schedules fire at UTC times rather than local times, log timestamps are UTC, date formatting in the app defaults to UTC unless explicitly converted.

Every service in the canonical shape sets `TZ`:

```yaml
environment:
  TZ: ${TZ:-Europe/London}
```

The format is an IANA timezone name (`Europe/London`, `America/New_York`, `Asia/Tokyo`), never an abbreviation like `BST` or `EST` — abbreviations are ambiguous and do not handle DST.

If a service is timezone-sensitive (cron containers especially), the `tzdata` package needs to be installed in the image. For `node:slim`: `RUN apt-get update && apt-get install -y --no-install-recommends tzdata && rm -rf /var/lib/apt/lists/*`. For distroless, the `:nonroot` tag may include it — verify before deploying.
