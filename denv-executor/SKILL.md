---
name: denv-executor
description: Set up and operate Magento 2, Shopify, PWA Studio, and Node.js development environments with the BSS denv CLI inside an isolated coding workspace.
version: 0.1.0
requires_tools:
  - enable_sandbox_tools
platforms:
  - linux
tags:
  - denv
  - docker
  - magento2
  - shopify
  - sandbox
category: commerce-development
---

# denv executor

Use this skill when a repository uses `.denv.env`, `.denv/`, or the task asks
you to create or operate a local environment with `denv`.

`denv` is a control-plane CLI around Docker Compose. It does not replace the
Docker daemon. Before running a mutating command, prove that both the CLI and
the daemon are available.

## Safety rules

- Work only inside the assigned workspace/repository.
- Never run `denv prune`, `docker system prune`, `denv down`, or `denv env
  remove` unless the task explicitly authorizes destructive cleanup.
- Never run `denv update`, `denv svc update`, or `denv env update`
  automatically in a normal coding task. These change shared templates/images.
- Never print or commit `~/.denv/auth`, API keys, access tokens, live
  `app/etc/env.php`, Shopify login state, or files under `~/.denv/secrets/`.
- Treat `.denv.env` and `.denv/` as source files: inspect their diff and remove
  production secrets before committing.
- Do not use an interactive command when an equivalent deterministic command or
  script exists.

## Mandatory preflight

Run from the repository root:

```bash
pwd
git status --short
denv version
docker version
docker compose version
docker info
test -d "${HOME}/.denv/svc"
```

Interpretation:

- `denv version` failing: the sandbox image is missing the denv binary.
- `docker version` showing only a client or `docker info` failing: no Docker
  daemon is reachable. Stop and report `blocked`; installing another Docker CLI
  does not fix this.
- A remote daemon must see the repository at the same absolute `WORK_DIR`.
  Verify the shared-volume contract. If the daemon cannot see the workspace,
  stop; denv bind mounts will otherwise point at a missing daemon-host path.
- `${HOME}/.denv/svc` missing: templates have not been provisioned. Run
  `denv svc init` only when credentials/network access were explicitly provided.
- Existing `.denv.env`: do not initialize again; inspect and validate it.

## Initialize a project without a TTY

`denv env init` prompts for the project name and can hang in an autonomous
shell tool. Prefer the bundled helper:

```bash
DENV_PROJECT_NAME="<safe-project-name>" denv-project-init
```

If the helper is unavailable:

```bash
printf '%s\n' "<safe-project-name>" | denv env init
```

After initialization:

```bash
test -f .denv.env
sed -n '1,220p' .denv.env
```

Set and verify at least:

```dotenv
DENV_PLATFORM=magento2  # or shopify, pwa-studio, nodejs, odoo
COMPOSE_PROJECT_NAME=<unique-job-or-project-slug>
VIRTUAL_HOST=<unique-local-host>
```

Do not add `DOCKER_UID`, `DOCKER_GID`, or `WORK_DIR` unless the installed denv
version requires an override; denv normally resolves them from the current
workspace. Infrastructure must mirror that resolved path into the Docker
daemon host or sidecar.

## Validate before starting containers

```bash
denv env debug
```

Review the rendered Compose configuration for:

- mounts that escape the assigned workspace;
- privileged mode or host devices;
- conflicting `COMPOSE_PROJECT_NAME` or hostnames;
- unexpected production credentials;
- services not needed by the task.

Only after validation:

```bash
denv env up
denv env ps
```

Use `denv env up -f` only when `.denv.env` or `.denv/docker-compose.yml`
changed and recreation is intentional.

## Run commands in project services

List services:

```bash
denv env ps
```

Magento defaults to the `cli` service:

```bash
denv shell cli
```

Because `denv shell` is interactive, autonomous agents should prefer a
non-interactive Docker Compose execution derived from `denv env debug`, or run a
single shell command through stdin:

```bash
printf '%s\n' 'php -v' 'exit' | denv shell cli
```

If this is unreliable, stop and request a dedicated non-interactive denv
`exec` capability instead of guessing input sequences.

## Platform notes

### Magento 2

- `DENV_PLATFORM=magento2`.
- Source is mounted into the Magento containers; run PHP/Composer/Magento
  commands in `cli`.
- Enable only required services (`DENV_REDIS`, search, RabbitMQ, Varnish, etc.).
- Use `.denv/env.php` only with local/sanitized values. Never copy production
  secrets unchanged.
- Run `denv env debug` before `up`, then deterministic gates inside `cli`.

### Shopify app

- `DENV_PLATFORM=shopify`.
- Source is mounted at `/app` in the Shopify service.
- `SHOPIFY_API_KEY` and `SHOPIFY_API_SECRET` are secrets; inject them at runtime
  and do not commit them to `.denv.env`.
- Shopify CLI login state belongs outside the repository. Treat it as a
  credential volume/secret.
- Prefer a development store and a development app configuration.

### Shopify theme

- Use the Shopify environment only when a long-running preview is required.
- Prefer deterministic commands such as `shopify theme check` and an
  unpublished/development theme push for CI-style jobs.
- Never publish or push to the live theme without explicit human approval.

## Completion evidence

Before reporting success, capture:

```bash
denv version
docker compose version
denv env debug
denv env ps
git status --short
git diff -- .denv.env .denv/
```

Report the platform, Compose project name, services started, verification
commands, preview URL if applicable, and any credentials or manual steps still
required.
