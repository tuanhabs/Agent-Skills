# Shopify Harness — Step-by-Step Build Guide

> Shopify implementation companion to `MAGENTO-2-HARNESS-GUIDE.md`.
> It supports theme, app/extension, and Admin API changes while keeping Git as
> the source of truth and production deployment human-gated.

## 0. Architecture

```text
shopify_dev → shopify_test → [cost gate] → shopify_review
     ▲             │                              │
     └── repair ───┘                              ▼
                                      [risk/HITL] → shopify_pr
```

Use two runtime modes:

| Mode | Use when | Runtime |
|---|---|---|
| **Code-only** | Theme/app edits, lint and unit tests | Lightweight Node/Shopify CLI sandbox |
| **denv** | Repository owns `.denv.env`, or a long-running proxied app preview is required | denv control image + isolated Docker daemon |

Do not use denv merely because it is available. A theme-check-only task should
not start a database, proxy, or nested service stack.

## 1. Intake classification

Every task must declare:

```json
{
  "change_type": "theme | app | extension | admin_data | headless",
  "environment_mode": "code-only | denv",
  "repo_url": "https://...",
  "base_branch": "main",
  "branch_slug": "short-slug",
  "shop_domain": "development-store.myshopify.com",
  "target_environment": "development",
  "ui_tests": true,
  "acceptance_criteria": []
}
```

Rules:

- `theme`: Liquid/JSON/CSS/JS; preview with a development or unpublished theme.
- `app`: backend/frontend application plus Shopify app configuration.
- `extension`: Admin, checkout, customer account, POS, or theme app extension.
- `admin_data`: GraphQL mutation changeset; do not mutate production in the
  inner coding loop.
- `headless`: Hydrogen/custom storefront plus Storefront API.

Missing store/environment/acceptance criteria returns `needs_clarification`.
Never infer the production shop or live theme ID.

## 2. Skills

Enable:

```text
shopify-executor
denv-executor        only for environment_mode=denv
```

The `denv-executor` pack lives at:

```text
implement/agent/skills/denv-executor/
```

Recommended `shopify-executor` contents:

```text
shopify-executor/
  SKILL.md
  references/
    theme-commands.md
    app-extension-commands.md
    admin-api-changesets.md
    review-checklist.md
    high-risk.md
```

## 3. Sandbox image

For denv jobs, build:

```text
implement/sandbox/denv-image/
```

The image contains denv, Docker Engine, and Compose in a privileged
single-container DinD sandbox. Do not run this image for untrusted multi-tenant
workloads on a shared worker.

Provision separately:

- `~/.denv/auth` and `~/.denv/svc`;
- Git credentials;
- Shopify CLI login/token or app credentials;
- development store identity.

Secrets must be injected into the sandbox/session and never committed to
`.denv.env`.

## 4. Shopify Dev prompt

```text
You are shopify_dev, an expert Shopify engineer in an isolated workspace.
Use shopify-executor for Shopify conventions. Load denv-executor only when the
input says environment_mode=denv or the repository already contains
.denv.env/.denv.

## INPUT
Parse the first user message as JSON. Required: change_type, repo_url,
base_branch, branch_slug, target_environment, acceptance_criteria.

## SETUP
1. Clone <repo_url> into `repo`.
2. Create `feature/<branch_slug>` from <base_branch>.
3. Inspect package/theme/app config before choosing commands.
4. Never use production credentials or a live theme.

If environment_mode=denv:
1. Run `verify-denv-runtime`; stop as blocked when `docker info` fails.
2. If `.denv.env` is absent:
   `cd repo && DENV_PROJECT_NAME=<branch_slug> denv-project-init`.
3. Ensure `DENV_PLATFORM=shopify`, a unique COMPOSE_PROJECT_NAME, and a unique
   VIRTUAL_HOST.
4. Run `cd repo && denv env debug`; inspect mounts, secrets and services.
5. Run `cd repo && denv env up`, then `cd repo && denv env ps`.
6. Never run denv prune/update/down without explicit authorization.

## IMPLEMENT
- Theme: smallest Liquid/section/block/assets change; preserve Theme Editor
  configurability.
- App/extension: use supported extension targets and Shopify APIs; do not patch
  Shopify Admin or checkout DOM.
- Admin data: create a versioned changeset with before/apply/rollback/verify;
  do not execute a production mutation.

## VERIFY
Theme:
  shopify theme check
  project JS lint/tests
  Playwright when ui_tests=true

App/extension:
  npm run lint
  npm run typecheck
  npm test
  shopify app build
  Playwright when ui_tests=true

Review git diff, commit only intended source/config files, push the feature
branch, and verify the remote branch exists.
```

## 5. Denv workflow for Shopify

The denv Shopify platform mounts source at `/app` inside its Shopify service.
Typical `.denv.env` values:

```dotenv
DENV_PLATFORM=shopify
COMPOSE_PROJECT_NAME=shopify-<job-id>
VIRTUAL_HOST=<job-id>.test
NODE_VERSION=20
NODE_COMMAND=
SHOPIFY_CLI_ENV=development
PORT=8081
```

Do not store these in Git:

```text
SHOPIFY_API_KEY
SHOPIFY_API_SECRET
access tokens
Shopify CLI login state
production database credentials
```

Operational sequence:

```bash
cd repo
verify-denv-runtime
denv env debug
denv env up
denv env ps
```

For a Shopify app, run setup/build commands in the Shopify service. Avoid
`denv shell` in autonomous runs when it opens an interactive shell; use a
deterministic command/exec integration or a bounded stdin script.

For theme preview, prefer Shopify CLI development/unpublished preview. The
preview URL and theme ID must be returned as evidence; publishing is forbidden
inside the AgentFlow.

## 6. Deterministic gates

Run cheap gates before LLM review:

```text
Theme:
  1. shopify theme check
  2. JSON/schema validation
  3. eslint/test
  4. Playwright opt-in
  5. secret scan

App/extension:
  1. lint
  2. typecheck
  3. unit/integration tests
  4. shopify app build
  5. Playwright opt-in
  6. secret scan
```

Failures route back to `shopify_dev` with exact command output and affected
files. Do not run expensive review after deterministic failure.

## 7. High-risk/HITL

Force human approval for:

- production Admin API mutations;
- checkout, payment, validation, shipping, discount, billing or auth changes;
- API scope expansion;
- webhook signing/processing changes;
- customer PII handling;
- publishing a theme or releasing an app version;
- deleting store data or removing rollback capability;
- adding/upgrading dependencies with material supply-chain impact.

The coding flow may create a PR and staging/development preview. Production
release belongs to the Saga outer loop after CI and human approval.

## 8. Git and deployment

```text
Sandbox:
  edit → deterministic gates → development preview → commit/push → PR

Saga:
  CI → preview URL → human approval → staging/release → verification
```

Git is the source of truth. Before changing a theme, detect drift between the
remote Shopify theme and the repository. Drift routes to clarification/sync,
not blind overwrite.

For Admin API changes, commit:

```text
shopify-changes/<job-id>/
  plan.json
  before.json
  apply.graphql
  rollback.graphql
  verification.json
```

## 9. Completion evidence

The final result must include:

- branch and commit;
- change type and risk;
- deterministic gate output;
- denv services/status when used;
- development store and preview URL/theme/app version;
- files changed;
- rollback procedure;
- explicit confirmation that production was not modified.
