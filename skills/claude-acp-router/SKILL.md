---
name: claude-acp-router
description: Route coding work to Claude ACP while keeping the sandbox as the authoritative workspace and applying changes deterministically.
version: 0.1.0
requires_tools:
  - enable_claude_acp_tool
  - enable_sandbox_tools
platforms:
  - linux
tags:
  - claude
  - acp
  - router
  - sandbox
category: coding-orchestration
---

# Claude ACP router

Use this workflow for code analysis, implementation, debugging and review.

## Ownership

- The sandbox is the only authoritative workspace.
- Claude ACP is the coding brain, but it cannot directly access the sandbox.
- You are the router and deterministic executor.
- Do not independently redesign or rewrite Claude's solution.
- Claude's host/control workspace is not a project checkout. If files appear
  there, treat them as non-authoritative mistakes unless they were explicitly
  provisioned instruction/memory files.

## Repository lifecycle in the sandbox

When the user task contains `repo_url`, `base_branch`, and `branch_slug`, the
Primary Agent must let Claude ACP reason about the development-environment
setup before implementation. Claude decides the setup plan and safe command
sequence; the Primary executes that sequence in OpenSandbox and returns
evidence.

The expected setup plan normally includes these sandbox commands from the active
sandbox workspace:

```bash
pwd
ls -la
test -d repo/.git || git clone --branch <base_branch> <repo_url> repo
cd repo && git fetch origin
cd repo && git checkout <base_branch>
cd repo && git pull --ff-only origin <base_branch>
cd repo && git checkout -B feature/<branch_slug>
cd repo && git status --short
ls -la repo
cd repo && test -f .denv.env && echo "denv project: yes" || echo "denv project: no"
cd repo && test -d .denv && find .denv -maxdepth 2 -type f -print | sort || true
```

Rules:

- Dotfiles are mandatory evidence. Plain `ls` hides `.denv.env` and `.denv`, so
  never conclude that denv is absent from plain directory listings.
- The canonical repository path is `<sandbox_workspace>/repo`, unless Claude's
  setup plan explicitly justifies another path and the Primary accepts it.
- All source paths sent to Claude and all returned edits must be scoped to the
  cloned repository. Prefer paths prefixed with `repo/` when using sandbox file
  tools from the workspace root.
- Never create application files directly under the sandbox workspace root, for
  example `/workspace/<agent>/Vendor/...` or `/workspace/<agent>/app/...`.
- Do not call host-side `git_add`, `git_commit`, or similar Git tools unless
  their working directory is explicitly configured to the same sandbox repo.
  For sandbox work, run Git through `swe_sandbox_shell`, e.g.
  `cd repo && git add ... && git commit ...`.
- The job is not complete merely because Claude proposed code. Completion
  requires sandbox evidence: `git status`, relevant tests/compile output, and
  either a local commit or a clear blocker explaining why commit/push could not
  be performed.

Claude ACP cannot inspect OpenSandbox directly. The Primary must read the
post-clone denv evidence from the sandbox and include it in the next packet.
If `.denv.env` or `.denv/` exists in `repo`, Claude's setup reasoning should
include the global-service and project-service sequence, usually:

```bash
cd repo && verify-denv-runtime
cd repo && sed -n '1,220p' .denv.env
cd repo && test -f .denv/docker-compose.yml && sed -n '1,220p' .denv/docker-compose.yml || true
test -d "${HOME}/.denv/svc" || denv svc init
denv svc up
cd repo && denv env debug
cd repo && denv env up
cd repo && denv env ps
```

If the Primary has not provided dotfile/denv evidence after clone, ask for that
evidence first. Do not let Claude implement or propose host package installs
from an incomplete packet.

If any setup step fails, the Primary sends Claude a `TURN_KIND:
environment_repair` packet with exact sanitized failure output. Claude may
propose one repair/diagnostic command sequence, or return `blocked`. Do not
ask Claude to implement from assumptions while the environment is blocked.

## Primary-to-Claude data contract

For every `claude_acp` call, build one self-contained context packet from the
current sandbox state. Never assume Claude remembers an earlier packet.
Never send placeholders such as `[Include contents from previous response]`,
`...`, `<same as above>`, or summaries pretending to be file contents. If exact
evidence is missing, collect it from sandbox first or mark that field as
`not_collected` and ask Claude for the next safe command sequence.

The packet must contain:

- `PACKET_VERSION`, stable `TASK_ID`, and `TURN_KIND`;
- verbatim task, platform, acceptance criteria, and constraints;
- repository state: `repo_url`, `base_branch`, `branch_slug`, current branch,
  commit SHA, and canonical repo path;
- post-clone dotfile evidence: `ls -la repo`, `.denv.env` presence, `.denv/`
  file list, and sanitized `.denv.env`/`.denv/docker-compose.yml` contents when
  present;
- `DEV_ENV_CONTEXT`, except during the first `environment_setup` turn, including
  whether repo clone, branch checkout, global denv services, and project denv
  services are ready;
- active skills, including `denv-executor` when denv applies;
- current Git status;
- exact source snapshots with relative paths and delimiters;
- sanitized search, runtime, test, and denv evidence;
- the JSON response schema below.

When denv applies, include:

```text
DEV_ENV_CONTEXT
status: ready | partial | blocked
sandbox_workspace: <active sandbox workspace>
canonical_repo_path: repo
repo_abs_path: <absolute path if known>
repo_url: <repo_url>
base_branch: <base_branch>
branch: feature/<branch_slug>
clone_status: ready | failed
checkout_status: ready | failed
denv_global_services: up | partial | skipped | failed
denv_project_services: up | skipped | failed
setup_evidence: <sanitized concise command output>

DENV_CONTEXT
present: true
platform: <safe DENV_PLATFORM value or unknown>
preflight: <sanitized denv/docker checks>
rendered_services: <focused denv env debug or ps output>
allowed_commands: denv env debug, denv env ps, safe tests/builds
forbidden_commands: denv prune/down/update/remove, docker system prune
```

Never transmit credentials, tokens, `~/.denv/auth`, `~/.denv/secrets`, or
unsanitized production configuration.

For the first call of a repository task, use:

```text
TURN_KIND: environment_setup
```

This first packet may omit source snapshots. It should include the raw task,
the active sandbox workspace, current `pwd`, `ls -la`, available tools, and any
known project conventions. If the repo has already been cloned, it must also
include `ls -la repo` and denv dotfile probes. Claude returns setup commands,
not code changes.

## Claude-to-Primary response contract

Require Claude to return one JSON object with no Markdown fence:

```json
{
  "contract_version": 1,
  "task_id": "same value received",
  "status": "ready | needs_context | blocked",
  "summary": "brief implementation summary",
  "assumptions": [],
  "required_context": [
    {
      "kind": "file | search | command",
      "value": "exact relative path, query, or safe command",
      "reason": "why it is required"
    }
  ],
  "setup_commands": [
    {
      "command": "safe non-interactive sandbox command",
      "purpose": "what this setup step establishes",
      "cwd": ".",
      "requires_approval": false
    }
  ],
  "changes": [
    {
      "path": "repo-relative/path or repo/path/from/workspace-root",
      "operation": "replace | create",
      "old_text": "exact existing text; empty for create",
      "new_text": "replacement or complete file",
      "reason": "why"
    }
  ],
  "verification_commands": [
    {
      "command": "safe non-interactive command",
      "purpose": "what it verifies",
      "requires_approval": false
    }
  ],
  "risks": [],
  "human_approval_required": false
}
```

Interpret responses strictly:

- `ready` with `TURN_KIND: environment_setup`: execute approved
  `setup_commands` in OpenSandbox, capture evidence, then send a normal
  implementation packet with `DEV_ENV_CONTEXT`.
- `ready` with implementation turns: validate and apply changes, then run
  approved verification.
- `needs_context`: apply nothing; collect only the requested context and resend
  a complete packet with `TURN_KIND: context_followup`.
- `blocked`: apply nothing; report the blocker with sandbox evidence.
- Invalid JSON, mismatched `task_id`, unsafe paths/commands, unknown operations,
  or missing exact `old_text`: reject and request one correction.
- A response claiming files were created, tests passed, or Git committed without
  sandbox evidence in the packet is invalid. Ask Claude to restate as proposed
  changes/commands only.

## denv coordination

Load and follow `denv-executor` whenever `.denv.env`, `.denv/`, or a denv
request is present. Claude should have the same skill installed in its control
workspace; naming a skill in the packet does not transmit its instructions.

Claude owns setup reasoning and command proposal. The Primary Agent owns command
execution, safety filtering, and evidence capture:

1. Ask Claude for `TURN_KIND: environment_setup` before implementation.
2. Execute approved repo clone/checkout and denv/Docker setup commands in the
   sandbox.
3. Sanitize results before sending `DEV_ENV_CONTEXT` and `DENV_CONTEXT`.
4. Reject interactive, destructive, credential-reading, update, and production
   commands unless explicitly authorized.
5. Prefer `denv svc up` before project services when global services are
   missing.
6. Prefer `denv env debug` before `denv env up`.
7. Treat sandbox output as authoritative over Claude's assumptions.

## Required sequence

1. Ensure an active sandbox session.
2. For tasks with repo metadata, send Claude `TURN_KIND: environment_setup`
   with the task and current sandbox workspace state.
3. Execute Claude's approved setup commands in the sandbox.
4. If setup fails, send `TURN_KIND: environment_repair` with exact output; stop
   if Claude returns `blocked`.
5. Inspect Git status, platform markers, and relevant skills inside the prepared
   repository.
6. Read the smallest relevant source and runtime context from the sandbox.
7. Send Claude a complete implementation packet with `DEV_ENV_CONTEXT` and the response
   contract.
8. If Claude requests context, collect only those items and call it again.
9. Validate every response field, path, operation, and command.
10. Apply exact edits in the sandbox repository.
11. Run Claude's verification commands after safety filtering.
12. If verification fails, send failure output, affected files, and Git diff in
    one `repair` packet.
13. Show Git diff, commit intended source files in `repo` when authorized, and
    report evidence. Push only when credentials and task scope allow it.

## Hard rules

- Never let Claude's host workspace replace sandbox state.
- Never ask Claude for implementation before the environment setup turn has
  completed or returned a clear blocker.
- Never apply an edit when `old_text` does not match exactly.
- Never create or edit a path outside the sandbox workspace.
- Never create project files outside `repo` when the task has `repo_url`.
- Never run destructive, deployment or production commands.
- Never silently improvise a code fix when Claude returns `needs_context`.
- Maximum two Claude calls for initial context and one repair call per user turn.
