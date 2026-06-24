# Magento 2 Harness — Step-by-Step Build Guide

> Implements the Magento 2 specialization of the coding-agent harness.
> It combines isolated sandboxes, deterministic gates, a native repair cycle,
> model tiering, Git handoff, and human approval for high-risk changes.

## 0. What you are building

```text
mage_dev → mage_test → [cost gate] → mage_review → [review gate] → mage_pr
    ▲          │                            │
    └── repair ┘                            └── repair → mage_dev
```

The pipeline uses four specialist agents:

| Agent | Alias | Responsibility |
|---|---|---|
| Magento Dev | `mage_dev` | Implement, validate, commit and push |
| Magento Test | `mage_test` | Run deterministic gates without modifying code |
| Magento Review | `mage_review` | Correctness, architecture and security review |
| Magento PR | `mage_pr` | Open the PR after all gates pass |

Magento-specific layers:

- `magento2-executor` Skill Pack for conventions and commands;
- optional `denv-executor` for a running Magento environment;
- cheap deterministic checks before expensive LLM review;
- repair back edges carrying structured findings;
- HITL for schema, data, ACL, core interception and dependency changes;
- Git and Composer credentials injected at sandbox bootstrap.

### 0.1 Two task modes

| Mode | Repository | Verification |
|---|---|---|
| **Module-only** | Magento module source | PHPCS, PHPStan, unit tests |
| **Full-app / denv** | Complete Magento project | Above plus runtime, DI compile and integration checks |

Module-only is the default. Do not start a full store merely because denv is
available.

### 0.2 Optional denv environment

Use denv when the repository contains `.denv.env`/`.denv`, or acceptance
criteria require a running Magento stack.

Before using it:

```bash
verify-denv-runtime
denv version
docker info
test -d "${HOME}/.denv/svc"
```

If `.denv.env` is absent:

```bash
cd repo
DENV_PROJECT_NAME="<job-or-project-slug>" denv-project-init
```

Required project identity:

```dotenv
DENV_PLATFORM=magento2
COMPOSE_PROJECT_NAME=<unique-job-slug>
VIRTUAL_HOST=<unique-job-host>
```

Validate before startup:

```bash
denv env debug
denv env up
denv env ps
```

Never run `denv prune`, `denv down`, `denv update`, `denv svc update`, or
`denv env update` in a normal coding task without explicit authorization.

## 1. Magento executor Skill Pack

Recommended layout:

```text
magento2-executor/
  SKILL.md
  references/
    cli-commands.md
    module-conventions.md
    coding-standards.md
    review-checklist.md
    high-risk.md
```

### Command order: cheap to expensive

```bash
phpcs --standard=Magento2 app/code/<Vendor>/<Module>
vendor/bin/phpstan analyse app/code/<Vendor>/<Module>
vendor/bin/phpunit <unit-test-path>
php bin/magento setup:di:compile
```

Only run the final command for a full Magento application when DI/config is
affected.

### Core conventions

- Never edit `vendor/magento`.
- Extend behavior in this order: plugin, observer, then preference.
- Inject dependencies through constructors; never use ObjectManager directly.
- Use declarative schema in `etc/db_schema.xml`.
- Treat schema and data patches as high risk.
- Admin controllers require routes, ACL and form-key protection.
- Escape all template output using the correct context-specific escaper.
- Do not commit `generated/`, `var/`, `vendor/`, `pub/static/` or
  `app/etc/env.php`.

### High-risk files and behavior

Force human review when the change touches:

- `etc/db_schema.xml`;
- `Setup/Patch/Schema` or `Setup/Patch/Data`;
- a `<preference>` overriding a core class;
- checkout, payment, sales, quote, customer or authorization interception;
- `etc/acl.xml` or admin authorization logic;
- dependency additions/upgrades in `composer.json`;
- environment/entry-point files under `app/etc` or `pub`.

Import `magento2-executor` for Dev, Test and Review. Import `denv-executor` only
for agents allowed to operate the full environment.

## 2. Shared pipeline contract

Every agent ends its response with a fenced JSON result:

```json
{
  "agent": "mage_dev | mage_test | mage_review | mage_pr",
  "status": "pass | fail | changes_requested | needs_clarification | no_tests | blocked",
  "branch": "feature/<branch_slug>",
  "repo_url": "<repo_url>",
  "risk": "low | high",
  "pushed": true,
  "findings": [
    {
      "severity": "critical | major | minor",
      "file": "repo/...",
      "line": 0,
      "evidence": "exact output or quote",
      "suggested_fix": "one sentence"
    }
  ],
  "evidence": {
    "phpcs": "",
    "phpstan": "",
    "phpunit": "",
    "di_compile": ""
  },
  "summary": "one sentence"
}
```

Rules:

- `status` is machine-readable and drives condition nodes.
- Only critical and major findings block.
- Minor findings are advisory.
- Human-readable text appears before the JSON block.
- The result block must be the final content in the message.

## 3. Agent configuration and prompts

### 3.1 `mage_dev`

Recommended model: a balanced coding model. Enable:

- Sandbox Tools;
- GitHub/GitLab tools as appropriate;
- `magento2-executor`;
- optionally `denv-executor`.

Prompt contract:

```text
You are an expert Magento 2 engineer working in an isolated sandbox.
Use magento2-executor for Magento conventions. Load denv-executor only when the
repository uses denv or the task requires a running store.

## INPUT
Parse the first user message as JSON:
- task_description
- repo_url
- base_branch
- branch_slug
- module
- environment_mode: module-only | denv
- acceptance_criteria
- constraints

## FIRST ATTEMPT
1. Clone <repo_url> into a directory literally named `repo`.
2. Create `feature/<branch_slug>` from <base_branch>.
3. Read only relevant files.
4. Prefer plugin > observer > preference.
5. Implement the smallest targeted change.
6. Run cheap gates first and repair failures before proceeding.
7. Review git status/diff.
8. Commit only intended source files.
9. Push the feature branch and verify the remote branch exists.

## REPAIR PASS
When input contains non-empty findings:
1. Reuse/check out the existing feature branch.
2. Read only the files named by findings.
3. Fix exactly the blocking findings.
4. Re-run the failed gates.
5. Commit and push on the same branch.

Do not stop after commit. Completion requires a verified remote branch.
Emit the shared RESULT envelope last.
```

### 3.2 `mage_test`

Use a low-cost model and read-only sandbox tools.

```text
You are a Magento 2 QA engineer. Never modify source files.
Use magento2-executor.

If the repository uses denv, first run:
  cd repo && denv env ps

Do not initialize, update, recreate, stop or prune denv from the Test stage.

Clone the pushed branch into `repo`, then run:
1. PHPCS
2. PHPStan when configured
3. PHPUnit when tests exist
4. setup:di:compile for an applicable full app

A missing optional tool/test is SKIPPED or NO_TESTS, not FAIL.
FAIL only when a gate actually ran and failed.
Return exact output and the shared RESULT envelope.
```

### 3.3 `mage_review`

Use the strongest review model and read-only tools.

```text
You are a senior Magento 2 reviewer.

Clone the pushed branch, inspect the complete diff and read every changed file.
Review:
- behavior and acceptance criteria;
- DI and extension mechanism;
- security, ACL, CSRF and escaping;
- backward compatibility;
- schema/data migration safety;
- tests and rollback;
- secret/generated-file leakage.

Only CRITICAL/MAJOR findings set status=changes_requested.
MINOR findings remain advisory.
Set risk=high when a high-risk rule is triggered.
Return the shared RESULT envelope.
```

### 3.4 `mage_pr`

Use a low-cost model and repository integration tools.

```text
You are a Magento release engineer. Do not write code.

Open a PR only when:
- mage_test.status is pass or no_tests;
- mage_review.status is pass;
- the branch was pushed;
- a required high-risk approval has been granted.

Use the dev summary for the title, include deterministic evidence and findings
in the body, and target the requested base branch.
Return PR URL in the shared RESULT envelope.
```

## 4. Deterministic gates

The Test stage should emit normalized results:

```json
{
  "coding_standard": "pass | fail | skipped",
  "static_analysis": "pass | fail | skipped",
  "unit_tests": "pass | fail | no_tests",
  "di_compile": "pass | fail | skipped"
}
```

Gate policy:

1. PHPCS failure blocks.
2. PHPStan failure blocks when configured.
3. PHPUnit failure blocks; absence becomes `no_tests`.
4. DI compile failure blocks for applicable full-app changes.
5. Do not spend an expensive review call while deterministic gates fail.

## 5. AgentFlow wiring

Recommended nodes:

```text
node_mage_dev
node_mage_test
node_cost_gate
node_mage_review
node_review_gate
node_risk_gate
node_mage_pr
```

### Cost gate

- source path: `status`;
- `fail` activates the repair edge to `node_mage_dev`;
- `pass` or `no_tests` proceeds to review.

### Review gate

- source path: `status`;
- `changes_requested` activates the repair edge;
- `pass` proceeds to the risk gate or PR.

On a repair edge, the condition node passes the upstream result unchanged.
`mage_dev` therefore receives findings, evidence, branch and repository URL.

Set a bounded `max_loop_iterations`—normally 3–5 for development—even if the
engine default is higher. When exhausted, route the outer CodingJob/Saga to
escalation instead of silently retrying.

## 6. Credentials

### Git

Configure repository credentials per agent. The sandbox bootstrap should inject:

- access token;
- Git username/name/email;
- HTTPS credential helper or SSH material.

Never put credentials in prompts.

### Composer / Magento Marketplace

For full-app `composer install`, inject Composer auth during session bootstrap:

```json
{
  "http-basic": {
    "repo.magento.com": {
      "username": "<MAGENTO_PUBLIC_KEY>",
      "password": "<MAGENTO_PRIVATE_KEY>"
    }
  }
}
```

Write this to `$COMPOSER_HOME/auth.json` inside the sandbox. Skip it for
module-only repositories that do not install Magento dependencies.

## 7. Human-in-the-loop policy

Use two layers:

1. Risk condition routes high-risk results to a human.
2. Tool Policy interrupts before external actions such as PR creation,
   production deployment or protected-branch push.

The coding harness may produce code, a branch, PR and development/staging
evidence. It must not silently deploy to production.

## 8. Cost, limits and observability

| Aspect | Recommendation |
|---|---|
| Repair depth | 3–5 loop iterations |
| Dev tool limits | Higher read/edit/shell budgets |
| Test tools | Read/shell only |
| Review tools | Read/search/diff only |
| PR tools | Repository API only |
| Cost | Skip review on deterministic failure |
| Context | Read only relevant files; spill large output |
| Observability | Persist node input/output and exact gate evidence |

## 9. Input example

```json
{
  "task_description": "Add an after-plugin on Product::getName to append a badge",
  "repo_url": "https://github.com/example/magento-module.git",
  "base_branch": "main",
  "branch_slug": "product-name-badge",
  "module": "Vendor_Module",
  "environment_mode": "module-only",
  "acceptance_criteria": [
    "Plugin is registered in etc/di.xml",
    "No ObjectManager usage",
    "Magento2 PHPCS passes",
    "A unit test covers the plugin"
  ],
  "constraints": [
    "Do not use a preference",
    "Do not change db_schema.xml"
  ]
}
```

## 10. Adoption order

1. Configure sandbox, four agents, skills, structured results and Git bootstrap.
2. Add deterministic gates and condition-based cost control.
3. Add bounded repair edges.
4. Add Composer credentials for full-app mode.
5. Add HITL and outer Saga/CodingJob escalation.
6. Tune model tiers, call limits and loop depth from observed runs.

The Shopify harness keeps the same orchestration shape; only the executor,
deterministic commands, credentials and high-risk policy change.
