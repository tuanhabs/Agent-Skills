---
name: denv-executor
description: Set up, inspect, start, and use BSS denv global services and per-project environments for Magento 2, Shopify, PWA Studio, Node.js, Odoo, and other Docker-based commerce workspaces.
version: 0.2.0
requires_tools:
  - enable_sandbox_tools
platforms:
  - linux
tags:
  - denv
  - docker
  - magento2
  - shopify
  - odoo
  - sandbox
category: commerce-development
---

# denv executor

Use this skill when a repository has `.denv.env`, `.denv/`, or the task asks
you to prepare a development environment with the BSS `denv` CLI.

`denv` has two layers:

1. **Global services**: `denv svc ...` manages shared infrastructure such as
   proxy, Adminer, Maildev/Mailpit, Elasticvue, hosts, and certificates under
   `${HOME}/.denv/svc`.
2. **Project services**: `denv env ...` manages the current repository's
   containers using that repo's `.denv.env` and `.denv/docker-compose.yml`.

Do not confuse them. `denv svc up` does not start the Magento/Odoo/Shopify
project. `denv env up` must be run from the project root that contains
`.denv.env`.

## Safety rules

- Work only inside the assigned workspace/repository.
- Never run `denv prune`, `docker system prune`, `denv down`, or `denv env
  remove` unless the task explicitly authorizes destructive cleanup.
- Never run `denv update`, `denv svc update`, or `denv env update`
  automatically in a normal coding task. These change shared templates/images.
- Never print or commit `${HOME}/.denv/auth`, API keys, access tokens, live
  `app/etc/env.php`, Shopify login state, or files under `${HOME}/.denv/secrets/`.
- Treat `.denv.env` and `.denv/` as source/config files: inspect their diff and
  remove production secrets before committing.
- Avoid interactive commands when a deterministic command exists.
- Do not install PHP, Composer, Node, databases, search engines, or other
  project runtimes directly on the sandbox host for normal project work. Use
  denv services. Installing host packages is sandbox-image maintenance and
  requires an explicit human request.

## Mandatory preflight

Two separate checks. Run them at different times.

### Preflight 1 — sandbox infra (run once before clone)

```bash
pwd
denv version
docker version
docker compose version
docker info
test -d "${HOME}/.denv/svc" && echo "denv svc templates: present" || echo "denv svc templates: missing"
```

Interpretation:

- `denv version` fails → sandbox image is missing the denv binary. Stop and report blocked.
- `docker info` fails → no Docker daemon is reachable. Stop and report blocked;
  installing another Docker CLI does not fix this.
- `denv svc templates: missing` → global service templates are not provisioned.
  Run `denv svc init` only if credentials/network access were provided.

### Preflight 2 — project denv probe (run after clone, inside repo)

See "Practical workflow — Step A". This is a separate step that must run
inside the cloned repo directory. Never skip it or substitute plain `ls`.

## Global services: `denv svc`

Global services are shared by project environments. They usually provide:

- `proxy` on HTTP/HTTPS routes;
- `adminer` for database browsing, often exposed on port `81`;
- `maildev`/`mailpit`, often exposed on port `82`;
- `elasticvue`, often exposed on port `83`;
- helper containers such as `hosts` and `mkcert`.

Bootstrap sequence:

```bash
test -d "${HOME}/.denv/svc" || denv svc init
denv svc up
docker ps
docker ps -a
```

Rules:

- `denv svc init` may require prior `denv auth ...`. If auth is missing, stop
  and ask for the denv bootstrap credential; do not invent or echo secrets.
- `denv svc up` can pull many images and may take minutes on a fresh sandbox.
- If `denv svc up` partially succeeds, inspect `docker ps -a`. Some helper
  containers may be `Created` while proxy/mail/adminer are already up.
- If Docker reports a cgroup error, `no space left on device`, or similar
  infrastructure failure, report the exact error and do not run destructive
  cleanup unless explicitly authorized.

## Project environment: `denv env`

Project services are controlled by the current repository's `.denv.env`.
Always prove you are in the project root:

```bash
pwd
ls -la
test -f .denv.env || { echo "Not in denv project root"; find . -maxdepth 3 -name .denv.env -print; exit 2; }
test -d .denv && find .denv -maxdepth 2 -type f -print | sort || true
sed -n '1,220p' .denv.env
test -f .denv/docker-compose.yml && sed -n '1,220p' .denv/docker-compose.yml || true
```

Important: `.denv.env` and `.denv/` are dotfiles. Plain `ls` hides them. Never
conclude that a cloned repo is not a denv project from plain `ls` output.

Before starting containers:

```bash
denv env debug
```

Review the rendered configuration for:

- mounts that escape the assigned workspace;
- unexpected privileged mode or host devices;
- conflicting `COMPOSE_PROJECT_NAME`;
- conflicting `VIRTUAL_HOST`;
- unexpected production credentials;
- services not required for the task.

Then start project services:

```bash
denv env up
denv env ps
docker ps
```

Use `denv env up -f` only when `.denv.env` or `.denv/docker-compose.yml`
changed and container recreation is intentional.

## Initialize a project without a TTY

If `.denv.env` is missing and the task asks you to create one, avoid the
interactive prompt:

```bash
DENV_PROJECT_NAME="<safe-project-name>" denv-project-init
```

If the helper is unavailable:

```bash
printf '%s\n' "<safe-project-name>" | denv env init
```

After initialization, inspect and set the minimum project identity:

```dotenv
DENV_PLATFORM=magento2  # or shopify, pwa-studio, nodejs, odoo
COMPOSE_PROJECT_NAME=<unique-job-or-project-slug>
VIRTUAL_HOST=<unique-local-host>
```

Do not add `DOCKER_UID`, `DOCKER_GID`, or `WORK_DIR` unless the installed denv
version requires an override. Denv normally resolves them from the current
workspace.

## Practical workflow for a cloned repo

For a task payload with `repo_url`, `base_branch`, and `branch_slug`:

```bash
mkdir -p /workspace/<agent-or-job>
cd /workspace/<agent-or-job>
git clone <repo_url> repo
cd repo
git checkout <base_branch>
git checkout -b feature/<branch_slug>
```

### Step A — probe for denv (always run first, inside repo)

`.denv.env` and `.denv/` are dotfiles. Plain `ls` hides them. You must use
explicit `test` probes. Run these from inside the cloned `repo` directory:

```bash
cd /workspace/<agent-or-job>/repo
ls -la
test -f .denv.env && echo "DENV_ENV=present" || echo "DENV_ENV=absent"
test -d .denv    && echo "DENV_DIR=present" || echo "DENV_DIR=absent"
```

**Decision gate — read the probe output before continuing:**

- `DENV_ENV=present` → this is a denv project. Continue to Step B.
- `DENV_ENV=absent`  → this is NOT a denv project. **Stop here.**
  Do not run `denv svc up`, `denv env up`, or any `denv env` command.
  Do not install PHP, Composer, Node, or other runtimes on the host.
  Report `denv_project: false` and wait for the next instruction.

Never infer denv absence from plain `ls` output. Only the explicit
`test -f .denv.env` probe is authoritative.

### Step B — bootstrap denv (only when DENV_ENV=present)

```bash
cd /workspace/<agent-or-job>/repo
find .denv -maxdepth 2 -type f -print | sort
sed -n '1,220p' .denv.env
test -f .denv/docker-compose.yml && sed -n '1,220p' .denv/docker-compose.yml || true
test -d "${HOME}/.denv/svc" || denv svc init
denv svc up
denv env debug
denv env up
denv env ps
docker ps
```

If `denv env up` reports it cannot read `.denv.env`, you are not in the
project root. Verify `pwd` and `cd` into the cloned repository.
If the repo has `.denv.env` but project services fail to start, do not fall
back to host PHP/composer/package installation. Report the exact denv/Docker
error and stop.

## Run commands inside project services

List project containers first:

```bash
denv env ps
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

Magento projects normally use the `cli` service for PHP, Composer, Magento CLI,
and PHPUnit:

```bash
printf '%s\n' 'php -v' 'exit' | denv shell cli
printf '%s\n' 'composer install' 'exit' | denv shell cli
printf '%s\n' 'php bin/magento setup:di:compile' 'exit' | denv shell cli
```

`denv shell <service>` may open an interactive shell. In autonomous runs, prefer
single-command stdin as above. If that is unreliable, stop and ask for a
dedicated non-interactive denv exec capability instead of guessing input
sequences.

## Project configuration patterns from real repos

Magento example shape:

```dotenv
DENV_PLATFORM=magento2
DENV_SEARCH_ENGINE=1
DENV_ADMINER=1
COMPOSE_PROJECT_NAME=v532
VIRTUAL_HOST=owen.test winny.test dunlop.test
PHP_VERSION=7.4
COMPOSER_VERSION=2.7
DATABASE_ENGINE=mariadb
DATABASE_ENGINE_VERSION=10.3
SEARCH_ENGINE=elasticsearch
SEARCH_ENGINE_VERSION=7.6
PHP_EXTENSIONS_ENABLE=imagick
```

Magento `.denv/docker-compose.yml` may mount local generated config:

```yaml
services:
  php:
    volumes:
      - ${WORK_DIR}/.denv/env.php:${WORK_DIR}/app/etc/env.php
  cli:
    volumes:
      - ${WORK_DIR}/.denv/env.php:${WORK_DIR}/app/etc/env.php
  nginx:
    volumes:
      - ${WORK_DIR}/.denv/nginx-sites.conf:/etc/nginx/templates/default.conf.template
```

Odoo example shape:

```dotenv
DENV_PLATFORM=odoo
COMPOSE_PROJECT_NAME=v573
VIRTUAL_HOST=v573.test
FORCE_HTTPS=true
ODOO_PYTHON_VERSION=3.12
ODOO_RC=/odoo/debian/odoo.conf
ODOO_DEV_FEATURES=reload,xml
ODOO_PROXY_MODE=true
POSTGRES_VERSION=16
POSTGRES_USER=odoo
```

Odoo `.denv/docker-compose.yml` may mount `.denv/odoo.conf` into `ODOO_RC`.

## Platform notes

### Magento 2

- `DENV_PLATFORM=magento2`.
- Source is mounted into project containers; run PHP/Composer/Magento commands
  in `cli`.
- Enable only needed services (`DENV_SEARCH_ENGINE`, `DENV_REDIS`,
  `DENV_RABBITMQ`, etc.).
- Use `.denv/env.php` only with local/sanitized values. Never copy production
  secrets unchanged.
- For module tasks, typical gates are:

```bash
printf '%s\n' 'composer install' 'php bin/magento setup:di:compile' 'exit' | denv shell cli
printf '%s\n' 'vendor/bin/phpunit -c dev/tests/unit/phpunit.xml.dist' 'exit' | denv shell cli
```

### Shopify app/theme

- `DENV_PLATFORM=shopify` when a long-running app/theme preview is required.
- Source is usually mounted at `/app` in the Shopify service.
- `SHOPIFY_API_KEY`, `SHOPIFY_API_SECRET`, and Shopify CLI login state are
  secrets; inject them at runtime and do not commit them.
- Prefer deterministic checks such as `shopify theme check`; never publish or
  push to a live theme without explicit approval.

### Odoo

- `DENV_PLATFORM=odoo`.
- Check `ODOO_RC`, PostgreSQL settings, and `.denv/odoo.conf` mounts.
- Run Odoo commands in the project service shown by `denv env ps`.

## Troubleshooting

- `open .../.denv.env: no such file or directory`: you are not in the project
  root. Find it with `find . -maxdepth 3 -name .denv.env -print`.
- `Error reading auth token`: run denv bootstrap/auth first, or ask for the
  credential. Do not print secrets.
- `no space left on device`: stop and report disk pressure. Use `docker system
  df` for evidence; destructive cleanup requires explicit approval.
- cgroup/threaded-mode errors during `denv svc up`: report the exact container
  and error. This is a Docker-in-Docker host/runtime limitation, not an app
  code issue.
- `denv shell` hangs: it likely opened an interactive shell. Use stdin command
  batches or request a non-interactive exec tool.

## Completion evidence

Before reporting success, capture:

```bash
denv version
docker compose version
test -d "${HOME}/.denv/svc" && echo "svc templates present"
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
denv env debug
denv env ps
git status --short
git diff -- .denv.env .denv/
```

Report the platform, `COMPOSE_PROJECT_NAME`, `VIRTUAL_HOST`, global services
started, project services started, verification commands, preview URLs/ports if
available, and any manual credential or infrastructure step still required.
