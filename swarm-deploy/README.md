# Docker Swarm Deploy

Deploys a stack to a remote Docker Swarm manager over SSH. Generic for any
project — only assumes you're deploying to Swarm. Logs in to the registry
locally (so `--with-registry-auth` can forward credentials to the Swarm
nodes), writes any file-based secrets your compose file needs (e.g. Vault
AppRole creds, or anything else that has to land on disk as a `file:`
secret), exports plain env vars for the compose file's own variable
substitution, then runs `docker stack deploy` against the remote daemon via
`DOCKER_HOST=ssh://...`.

## Why `DOCKER_HOST` over SSH instead of rendering + `scp`/remote script

Because the `docker` CLI runs locally on the GitHub Actions runner (just
talking to a *remote* daemon over an SSH tunnel), the compose file's native
`${VAR}` substitution is resolved from the **job's own environment** —
there is no separate "render the template with envsubst, base64-encode it,
ship it to the host, decode it there" dance, and no remote script to keep
in sync. Whatever env vars are present in the job when `docker stack
deploy` runs is what the compose file sees. This also means secrets never
have to be written to disk on the remote host — only (temporarily) on the
GitHub Actions runner, which is torn down after the job.

## Naming convention

The stack name follows the `env-slug` + `app-name` convention:

- **Stack name**: `<env-slug>-<app-name>` (e.g. `acme-crm-prod-api`),
  computed for you and exposed as the `stack-name` output.
- **In-compose service name**: recommended plain `<app-name>` (e.g. `api`,
  `web`) — i.e. your compose file's `services:` key is just `api` or `web`.
  With this, the **full Swarm service id** (what `docker service ls`/
  `docker service inspect` shows) is `<stack-name>_<app-name>`, e.g.
  `acme-crm-prod-api_api` — computed for you as the `service-id` output and
  used internally for the convergence check.
- You are **not required** to name the service plainly. `STACK_NAME` is a
  real environment variable during the deploy step (see below), so a
  compose file can equally use the older
  `services: ${STACK_NAME}-api:` style (service key itself includes the
  stack name) — see "Legacy-style service naming" further down. If you do
  that, set the `service-name` input so the convergence check looks at the
  right service id.

This keeps stack names descriptive (project + environment + app) while
keeping a plainly-named compose file portable across environments (it
never needs to know its own stack name).

## `STACK_NAME` and `IMAGE` are real env vars at deploy time — use them anywhere

The "Deploy stack" step runs `docker stack deploy` with `STACK_NAME` and
`IMAGE` set as actual process environment variables (not just step
outputs), in addition to anything you passed via `secret-files` or
`env-vars` (those are exported earlier via `$GITHUB_ENV`, so they're also
present by the time this step runs). Since Docker's compose-file variable
substitution resolves `${VAR}` against the environment of whatever process
invokes `docker stack deploy`, **all of these are available anywhere in the
compose file** — not just in `image:` — including:

- A service's `environment:` list (`- DATABASE_URL=${DATABASE_URL}`)
- A `deploy.labels` entry (`"traefik.http.routers.${STACK_NAME}.rule=..."`)
- A `networks.*.aliases` entry (`aliases: [${STACK_NAME}]`)
- Even a service **key** itself (`services: ${STACK_NAME}-api:`)

See "Passing many app env vars + legacy-style service naming + network
alias" below for a complete example combining all of these.

## What it does, step by step

1. **Compute stack and service names** — `env-slug` + `app-name` (or your
   `stack-name` override), exposed as outputs.
2. **Configure SSH access** — starts an `ssh-agent`, adds `ssh-key`, and
   writes `~/.ssh/known_hosts` from `ssh-known-hosts` if you passed it, or
   populates it via `ssh-keyscan` against `ssh-host:ssh-port` otherwise.
3. **Write file-based secrets** (only if `secret-files` is non-empty) —
   for each `NAME=VALUE` line, writes `VALUE` to a private (`chmod 600`)
   temp file under `$RUNNER_TEMP` and exports `NAME=<path to that file>`
   into `$GITHUB_ENV`, so your compose file's `secrets:` section can
   reference `file: ${NAME}`.
4. **Set extra env vars** (only if `env-vars` is non-empty) — exports each
   `KEY=VALUE` line into `$GITHUB_ENV` as-is, for any plain (non-secret)
   value your compose file substitutes (domain, port, feature flag, etc.).
5. **Log in to the registry** locally on the runner (`docker login`) — this
   is what lets `--with-registry-auth` forward a signed, scoped auth token
   to the Swarm nodes so they can pull a private image.
6. **Deploy the stack** — `docker stack deploy --compose-file <file>
   [--with-registry-auth] [--prune] <stack-name>` with `DOCKER_HOST`
   pointed at the Swarm manager over SSH. `IMAGE` is exported for this step
   so the compose file can reference `image: ${IMAGE}`.
7. **Verify convergence** (unless `verify-convergence: false`) — inspects
   the deployed service and compares its running image against the one you
   asked to deploy. If they don't match, the step fails loudly and dumps
   `docker service ps` output. This catches a case that would otherwise go
   unnoticed: Swarm's `update_config.failure_action: rollback` silently
   reverting a bad deploy back to the old image while `docker stack deploy`
   itself still reports success.
8. **Clean up credentials** (`if: always()`) — removes the temp secret
   files, `known_hosts`, and kills the `ssh-agent` process, regardless of
   whether the deploy succeeded.

## Usage

### Basic — single app, no file-based secrets

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image: ${{ steps.build.outputs.image }}
    steps:
      - uses: alexandergv2117/actions/docker-build-push@main
        id: build
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          app-name: api
          dockerfile: apps/api/Dockerfile

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: prod
    steps:
      - uses: actions/checkout@v4

      - uses: alexandergv2117/actions/swarm-deploy@main
        id: deploy
        with:
          compose-file: infra/deploy/swarm/api.deployment.yml
          env-slug: acme-crm-prod
          app-name: api
          image: ${{ needs.build.outputs.image }}
          registry: ghcr.io
          registry-username: ${{ github.actor }}
          registry-password: ${{ secrets.GITHUB_TOKEN }}
          ssh-host: ${{ secrets.SWARM_SSH_HOST }}
          ssh-user: ${{ secrets.SWARM_SSH_USER }}
          ssh-key: ${{ secrets.SWARM_SSH_KEY }}
          env-vars: |
            HOST=${{ vars.API_DOMAIN }}
```

`ssh-known-hosts` is optional and omitted here — the action populates
`known_hosts` for you via `ssh-keyscan` against `ssh-host:ssh-port`. See
"Passing many app env vars + legacy-style service naming + network alias"
further down for an example that pins it instead.

Matching compose file (`infra/deploy/swarm/api.deployment.yml`):

```yaml
networks:
  traefik_traefik_proxy:
    external: true
  app-network:
    external: true

services:
  api:
    image: ${IMAGE}
    environment:
      - NODE_ENV=production
    networks:
      traefik_traefik_proxy:
      app-network:
    deploy:
      replicas: 1
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.${STACK_NAME}.rule=Host(`${HOST}`)"
```

`${STACK_NAME}` in the Traefik label above works out of the box — no extra
wiring needed. The action sets `STACK_NAME` as a real environment variable
during the "Deploy stack" step itself, which is exactly when `docker stack
deploy` parses the compose file and resolves its `${VAR}` substitutions.

### With file-based secrets (e.g. Vault AppRole creds)

```yaml
- uses: alexandergv2117/actions/swarm-deploy@main
  with:
    compose-file: infra/deploy/swarm/api.deployment.yml
    env-slug: acme-crm-prod
    app-name: api
    image: ${{ needs.build.outputs.image }}
    registry: ghcr.io
    registry-username: ${{ github.actor }}
    registry-password: ${{ secrets.GITHUB_TOKEN }}
    ssh-host: ${{ secrets.SWARM_SSH_HOST }}
    ssh-user: ${{ secrets.SWARM_SSH_USER }}
    ssh-key: ${{ secrets.SWARM_SSH_KEY }}
    secret-files: |
      VAULT_ADDR_FILE=${{ secrets.VAULT_ADDR }}
      VAULT_SECRET_PATH_FILE=${{ secrets.VAULT_SECRET_PATH }}
      VAULT_APPROLE_MOUNT_FILE=${{ secrets.VAULT_APPROLE_MOUNT }}
      VAULT_ROLE_ID_FILE=${{ secrets.VAULT_ROLE_ID }}
      VAULT_SECRET_ID_FILE=${{ secrets.VAULT_SECRET_ID }}
```

Matching compose file:

```yaml
secrets:
  vault_addr:
    file: ${VAULT_ADDR_FILE}
  vault_secret_path:
    file: ${VAULT_SECRET_PATH_FILE}
  vault_approle_mount:
    file: ${VAULT_APPROLE_MOUNT_FILE}
  vault_role_id:
    file: ${VAULT_ROLE_ID_FILE}
  vault_secret_id:
    file: ${VAULT_SECRET_ID_FILE}

services:
  api:
    image: ${IMAGE}
    secrets:
      - vault_addr
      - vault_secret_path
      - vault_approle_mount
      - vault_role_id
      - vault_secret_id
```

Your app then reads these at startup from
`/run/secrets/vault_addr`, `/run/secrets/vault_role_id`, etc. (Swarm's
standard secret mount path) — typically to bootstrap a Vault client and
pull the *real* application secrets (DB URL, API keys, ...) from Vault at
runtime, rather than baking them into the stack file or the image.

### Passing many app env vars + legacy-style service naming + network alias

Two patterns that come up together in practice:

1. A service that needs a long list of plain (non-file) secrets/config
   injected as `environment:` entries (e.g. an API with a DB URL, auth
   secrets, third-party API keys, ...) — all of these are just `env-vars`
   lines, however many you need.
2. Services named after the stack itself (`${STACK_NAME}-api`) instead of
   plainly (`api`), and/or a plainly-named service that still needs to be
   reachable by other services *via* the full stack name — done with a
   `networks.*.aliases` entry.

```yaml
- uses: alexandergv2117/actions/swarm-deploy@main
  id: deploy-api
  with:
    compose-file: .deploy/swarm/api.prod.template.yml
    env-slug: acme-crm
    app-name: api
    # the compose file below names the service `${STACK_NAME}-api`, not
    # plain `api` — point the convergence check at the right service id:
    service-name: acme-crm-api-api
    image: ${{ needs.build.outputs.image }}
    registry: ghcr.io
    registry-username: ${{ github.actor }}
    registry-password: ${{ secrets.GITHUB_TOKEN }}
    ssh-host: ${{ secrets.SSH_HOST }}
    ssh-user: ${{ secrets.SSH_USER }}
    ssh-key: ${{ secrets.SSH_KEY }}
    env-vars: |
      BETTER_AUTH_SECRET=${{ secrets.BETTER_AUTH_SECRET }}
      DATABASE_URL=${{ secrets.DATABASE_URL }}
      BETTER_AUTH_URL=${{ secrets.BETTER_AUTH_URL }}
      FRONTEND_URL=${{ secrets.FRONTEND_URL }}
      STRIPE_SECRET_KEY=${{ secrets.STRIPE_SECRET_KEY }}
      STRIPE_WEBHOOK_SECRET=${{ secrets.STRIPE_WEBHOOK_SECRET }}
      RESEND_API_KEY=${{ secrets.RESEND_API_KEY }}
```

Matching compose file — the service key itself is `${STACK_NAME}-api`,
every value in `environment:` comes straight from an `env-vars` line, and
`${IMAGE}` / `${STACK_NAME}` resolve the same way they always do:

```yaml
services:
  ${STACK_NAME}-api:
    image: ${IMAGE}
    environment:
      - BETTER_AUTH_SECRET=${BETTER_AUTH_SECRET}
      - DATABASE_URL=${DATABASE_URL}
      - BETTER_AUTH_URL=${BETTER_AUTH_URL}
      - FRONTEND_URL=${FRONTEND_URL}
      - STRIPE_SECRET_KEY=${STRIPE_SECRET_KEY}
      - STRIPE_WEBHOOK_SECRET=${STRIPE_WEBHOOK_SECRET}
      - RESEND_API_KEY=${RESEND_API_KEY}
    deploy:
      replicas: 2
```

And a second, plainly-named service (`web`) in the same or a different
stack that other services should still be able to reach *by the full stack
name* — e.g. because a shared reverse-proxy or another stack's service
resolves it that way. `aliases:` adds `${STACK_NAME}` as an extra DNS name
for the service on that network, on top of its real service name. This
example also shows the optional `ssh-known-hosts` input pinned (rather than
left empty for the `ssh-keyscan` fallback):

```yaml
- uses: alexandergv2117/actions/swarm-deploy@main
  id: deploy-web
  with:
    compose-file: infra/deploy/swarm/web.deployment.yml
    env-slug: acme-crm-prod
    app-name: web
    # service key is plain `web`, so the default service-name (= app-name)
    # is already correct here — no service-name override needed
    image: ${{ needs.build.outputs.image }}
    registry: ghcr.io
    registry-username: ${{ github.actor }}
    registry-password: ${{ secrets.GITHUB_TOKEN }}
    ssh-host: ${{ secrets.SWARM_SSH_HOST }}
    ssh-user: ${{ secrets.SWARM_SSH_USER }}
    ssh-key: ${{ secrets.SWARM_SSH_KEY }}
    # ssh-known-hosts is optional — pass it when you want strict host-key
    # verification instead of the default ssh-keyscan fallback (generate
    # one with `ssh-keyscan -p <port> <host>` and store it as a secret):
    ssh-known-hosts: ${{ secrets.SWARM_SSH_KNOWN_HOSTS }}
    secret-files: |
      VAULT_ADDR_FILE=${{ secrets.VAULT_ADDR }}
      VAULT_SECRET_PATH_FILE=${{ secrets.VAULT_SECRET_PATH }}
      VAULT_APPROLE_MOUNT_FILE=${{ secrets.VAULT_APPROLE_MOUNT }}
      VAULT_ROLE_ID_FILE=${{ secrets.VAULT_ROLE_ID }}
      VAULT_SECRET_ID_FILE=${{ secrets.VAULT_SECRET_ID }}
```

```yaml
services:
  web:
    image: ${IMAGE}
    environment:
      - PORT=3000
      - NODE_ENV=production
    secrets:
      - vault_addr
      - vault_secret_path
      - vault_approle_mount
      - vault_role_id
      - vault_secret_id
    networks:
      traefik_traefik_proxy:
      app-network:
        aliases:
          - ${STACK_NAME}
```

Here `web` is the service's real (plain) name — so `service-name` doesn't
need to be set, the default (`app-name` → `web`) is already correct for the
convergence check — while `${STACK_NAME}` (e.g. `acme-crm-prod-web`) is
*additionally* resolvable on `app-network`, letting e.g. an `api` service
in another stack reach it at `http://acme-crm-prod-web:3000` without
hardcoding the real service name.

### Full pipeline — build, deploy, notify

```yaml
name: Deploy API to PROD

on:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    environment: prod
    outputs:
      image: ${{ steps.build.outputs.image }}
    steps:
      - uses: alexandergv2117/actions/docker-build-push@main
        id: build
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          app-name: api
          dockerfile: apps/api/Dockerfile

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: prod
    steps:
      - uses: actions/checkout@v4
      - uses: alexandergv2117/actions/swarm-deploy@main
        id: deploy
        with:
          compose-file: infra/deploy/swarm/api.deployment.yml
          env-slug: acme-crm-prod
          app-name: api
          image: ${{ needs.build.outputs.image }}
          registry: ghcr.io
          registry-username: ${{ github.actor }}
          registry-password: ${{ secrets.GITHUB_TOKEN }}
          ssh-host: ${{ secrets.SWARM_SSH_HOST }}
          ssh-user: ${{ secrets.SWARM_SSH_USER }}
          ssh-key: ${{ secrets.SWARM_SSH_KEY }}
          env-vars: |
            HOST=${{ vars.API_DOMAIN }}

  notify:
    needs: [build, deploy]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - uses: alexandergv2117/actions/slack-notify@main
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
          status: ${{ needs.deploy.result }}
          environment: prod
          action-name: Deploy
          extra-fields: '[{"label":"Image","value":"${{ needs.build.outputs.image }}"},{"label":"Stack","value":"${{ needs.deploy.outputs.stack-name }}"}]'
```

### Skipping the convergence check

Useful for a first-ever deploy of a stack (nothing to compare against yet
in some edge cases) or for services you don't want to gate the workflow on:

```yaml
- uses: alexandergv2117/actions/swarm-deploy@main
  with:
    # ...
    verify-convergence: "false"
```

### Non-default SSH port, no prune, extra raw flags

```yaml
- uses: alexandergv2117/actions/swarm-deploy@main
  with:
    # ...
    ssh-port: "2222"
    prune: "false"
    extra-args: "--resolve-image changed"
```

## Inputs

| Input                 | Required | Default | Description |
|------------------------|----------|---------|-------------|
| `compose-file`         | yes      | —       | Path to the stack/compose file, relative to the repo root (e.g. `infra/deploy/swarm/api.deployment.yml`). The caller must check out the repo first — this action does not do it for you. |
| `env-slug`             | yes      | —       | Project + environment slug (e.g. `acme-crm-prod`). Combined with `app-name` to form the stack name. |
| `app-name`             | yes      | —       | App name (e.g. `api`, `web`). Combined with `env-slug` to form the stack name, and expected to match the service name inside the compose file. |
| `stack-name`           | no       | `""`    | Override the computed stack name (`env-slug-app-name`) entirely. Use this only if you need a naming scheme that doesn't fit the convention. |
| `service-name`         | no       | `""`    | Name of the service key inside the compose file, used only to build the service id for the convergence check (`stack-name_service-name`). Defaults to `app-name`, which matches a plainly-named service (`services: api:`). Set this explicitly when the compose file names the service after the stack itself (`services: ${STACK_NAME}-api:`) — pass the literal rendered value, e.g. `"acme-crm-prod-api-api"`. |
| `image`                | yes      | —       | Full image ref to deploy (e.g. from `docker-build-push`'s `image` output). Exported as `IMAGE` for the compose file's variable substitution. |
| `registry`             | no       | `"ghcr.io"` | Container registry host. |
| `registry-username`    | yes      | —       | Registry username, used for the local `docker login` that backs `--with-registry-auth`. |
| `registry-password`    | yes      | —       | Registry password or token. **Pass as a secret.** |
| `ssh-host`             | yes      | —       | Swarm manager SSH host. |
| `ssh-user`             | yes      | —       | Swarm manager SSH user. |
| `ssh-port`             | no       | `"22"`  | Swarm manager SSH port. |
| `ssh-key`              | yes      | —       | Private key for SSH access to the Swarm manager. **Pass as a secret.** |
| `ssh-known-hosts`      | no       | `""`    | Known-hosts entry for the Swarm manager. Leave empty to populate it automatically via `ssh-keyscan` instead (slightly less safe against a MITM on the very first connection, but one less secret to manage — generate and pin one with `ssh-keyscan -p <port> <host>` if you want strict verification). |
| `secret-files`         | no       | `""`    | File-based secrets the compose file references via `file: ${SOME_FILE}` (e.g. Vault AppRole creds). One `NAME=VALUE` per line; each is written to a private (`chmod 600`) temp file under `$RUNNER_TEMP`, and `NAME` is exported pointing at that file's path. |
| `env-vars`             | no       | `""`    | Extra plain env vars the compose file's variable substitution needs (domain, feature flags, non-secret config). One `KEY=VALUE` per line. |
| `with-registry-auth`   | no       | `"true"`| Pass `--with-registry-auth` to `docker stack deploy`, forwarding the local `docker login` credentials to the Swarm nodes so they can pull a private image. |
| `prune`                | no       | `"true"`| Pass `--prune` to `docker stack deploy` — removes services defined in a previous version of the stack but no longer present in the compose file. |
| `verify-convergence`   | no       | `"true"`| After deploying, check that the service actually ends up running the expected image. Catches a silent Swarm rollback (`update_config.failure_action: rollback`) that would otherwise leave the old image running while the workflow still reports success. |
| `extra-args`           | no       | `""`    | Extra raw arguments appended to the `docker stack deploy` command (e.g. `--resolve-image changed`). Split on whitespace — don't use this for values containing spaces. |

## Outputs

| Output        | Description |
|---------------|-------------|
| `stack-name`  | Computed (or overridden) stack name, e.g. `acme-crm-prod-api`. |
| `service-id`  | Full Swarm service id (`stack-name_service-name`, `service-name` defaulting to `app-name`), e.g. `acme-crm-prod-api_api`. Useful if you want to run your own extra `docker service ...` checks in a later step. |

## Secrets and permissions this action needs

Typically wired up once per environment as GitHub Environment secrets
(`environment: prod` on the job), so they're protected by environment
protection rules:

- `SWARM_SSH_HOST`, `SWARM_SSH_USER`, `SWARM_SSH_KEY` (and ideally
  `SWARM_SSH_KNOWN_HOSTS`) — SSH access to a Swarm **manager** node (not a
  worker; `docker stack deploy` requires manager privileges).
- A registry credential pair with push/pull rights — `GITHUB_TOKEN` works
  out of the box for GHCR within the same repo (needs `packages: write` on
  the build job and `packages: read` is enough on the deploy job, since it
  only needs to pull).
- Whatever your `secret-files`/`env-vars` reference (Vault creds, domain
  names, API keys the compose file needs, ...).

## Requirements

- The Swarm manager must already be initialized (`docker swarm init`), and
  any external networks the compose file references (e.g.
  `traefik_traefik_proxy`, `app-network`) must already exist on that Swarm
  — this action does not create them.
- `ssh`, `ssh-agent`, `ssh-keyscan` and `docker` (client) must be available
  on the runner — all preinstalled on `ubuntu-latest`. On a self-hosted
  runner, make sure they're installed.
- The SSH user must be able to run `docker` commands on the manager
  (typically a member of the `docker` group, or root).

## Notes and gotchas

- **Don't `envsubst` the compose file yourself before calling this
  action.** Variable substitution happens natively when `docker stack
  deploy` parses the file, using whatever's in the job's environment at
  that point (including anything this action wrote via `secret-files` /
  `env-vars`). Pre-rendering it yourself is redundant and, if you then pass
  a fully-rendered file, harmless but pointless.
- **`secret-files` and `env-vars` are exported via `$GITHUB_ENV`**, which
  means they leak into every subsequent step of the *calling job* (not
  just this action's own remaining steps) — same as the Vault-file pattern
  this action is modeled on. This is intentional and usually harmless, but
  keep it in mind if a later step in the same job is security-sensitive
  about its environment.
- **Order inside `secret-files`/`env-vars` doesn't matter**, but each line
  must be a single `NAME=VALUE` pair; only the *first* `=` is treated as
  the separator, so values containing `=` (base64, JSON, URLs with query
  strings) are safe.
- **The convergence check defaults to a service id of
  `<stack-name>_<app-name>`** — i.e. that your compose file's service key
  is exactly `app-name`. If your compose file names the service differently
  (e.g. the legacy `${STACK_NAME}-api` style), set `service-name` to the
  literal rendered service key so the check looks at the right service id,
  instead of disabling `verify-convergence` altogether.
- **This action always deploys over SSH to a single manager.** It doesn't
  support deploying through a load balancer in front of multiple managers,
  or non-SSH transports (e.g. an exposed TCP Docker socket) — if you need
  that, invoke `docker stack deploy` yourself with a different `DOCKER_HOST`.
- **Secrets written to `$RUNNER_TEMP` are cleaned up in an `if: always()`
  step**, but if the job is cancelled hard (not just failed) cleanup may
  not run — GitHub-hosted runners are ephemeral and destroyed after the
  job regardless, so this is a defense-in-depth measure rather than a
  strict requirement.
