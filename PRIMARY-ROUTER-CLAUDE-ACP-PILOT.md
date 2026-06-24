# Pilot: Primary LangChain Agent as Router, Claude ACP as Coding Brain

This guide tests the architecture without changing Agent Manager, Claude ACP,
OpenSandbox, or adding an MCP bridge.

The pilot uses only existing prompts, skills, tools, and configuration.

## 1. Goal and boundary

Target behavior:

```text
User
  ↓
Primary LangChain Agent
  ├── creates/reuses the sandbox
  ├── reads the relevant source and runtime evidence
  ├── asks Claude ACP to reason and produce a structured changeset
  ├── applies the changeset with swe_sandbox_* tools
  ├── runs verification in the sandbox
  └── asks Claude ACP for one repair changeset when verification fails
```

Responsibilities:

| Component | Responsibility |
|---|---|
| Primary Agent | Routing, sandbox lifecycle, context collection, deterministic edits, commands, verification and reporting |
| Claude ACP | Analysis, architecture decisions, code generation, review and repair recommendations |
| Sandbox | Authoritative filesystem, Git state and execution environment |
| Git | Durable code checkpoint and final diff |

Important limitation:

> Claude ACP is not connected to the OpenSandbox filesystem in this pilot.

Claude runs in its configured host workspace. It must not assume that its local
files are the sandbox files. The Primary Agent sends relevant source snapshots
to Claude and applies Claude's structured changeset inside the sandbox.

This pilot is suitable for small, focused changes. It is not a replacement for
a native sandbox bridge or running Claude inside the sandbox.

## 2. Why use a structured changeset?

Do not ask Claude to return an essay followed by arbitrary code blocks. Use a
machine-readable contract:

```json
{
  "status": "ready | needs_context | blocked",
  "summary": "short explanation",
  "assumptions": [],
  "required_context": [],
  "changes": [
    {
      "path": "relative/path/to/file",
      "operation": "replace | create",
      "old_text": "exact existing text for replace",
      "new_text": "replacement or complete new file content",
      "reason": "why this change is needed"
    }
  ],
  "verification_commands": [],
  "risks": [],
  "human_approval_required": false
}
```

Benefits:

- the Primary Agent can reject paths outside the workspace;
- `replace` supports exact-match edits through `swe_sandbox_edit_file`;
- `create` supports deterministic writes through `swe_sandbox_write_file`;
- missing context is explicit;
- verification commands can be filtered before execution;
- Claude does not need direct sandbox access.

## 3. Create a dedicated pilot Agent

Create a new Agent rather than changing a production coding agent.

Suggested values:

```text
Name: Claude ACP Router Pilot
Alias: claude_acp_router_pilot
```

Use a low-cost primary model that supports reliable tool calling. The primary
model should not be responsible for architecture or code generation.

Recommended primary settings:

```text
Temperature: 0
Model: a small/cheap tool-calling model
Max output: sufficient for tool arguments and short status messages
```

## 4. Enable only the required tools

Enable:

```text
AI Code: Claude ACP agent
Sandbox Tools
Skill Packs or Skills
```

For the first pilot, enable these sandbox tools:

```text
swe_sandbox_session_status
swe_sandbox_session_create
swe_sandbox_session_restart
swe_sandbox_shell
swe_sandbox_read_file
swe_sandbox_write_file
swe_sandbox_edit_file
swe_sandbox_list_files
swe_sandbox_find_files
swe_sandbox_search
swe_sandbox_git_diff
```

Do not enable repository push, PR, deployment, production mutation, or sandbox
termination until the read/edit/test loop is stable.

### Disable AI Code output passthrough

Disable:

```text
AI Code tool output passthrough
```

The middleware normally returns Claude ACP's output directly and ends the graph
when a turn calls only an AI Code tool. This pilot needs the Primary Agent to
continue after Claude returns so it can apply the changeset to the sandbox.

## 5. Configure Claude ACP

Suggested configuration:

```text
Runtime command: npx
Command args: -y @agentclientprotocol/claude-agent-acp
Timeout: 600 seconds
Keep ACP session alive per conversation: enabled
Auto-approve: disabled initially
Anthropic API key: empty when using an existing Claude subscription login
```

Set the shared AI Code app directory to a dedicated host directory:

```text
/app/workspaces/claude_acp_router_pilot/brain
```

Do not point it at the Agent Manager source tree or another real project. This
directory is a control workspace for Claude instructions, not the authoritative
repository.

Create it on the Agent Manager host/container:

```bash
mkdir -p /app/workspaces/claude_acp_router_pilot/brain/.claude/skills/coding-brain
```

## 6. Configure Claude's own instructions

Create this file in Claude's host control workspace:

```text
/app/workspaces/claude_acp_router_pilot/brain/CLAUDE.md
```

Suggested content:

```markdown
# Role: Remote coding brain

You are the coding and review brain behind a separate orchestration agent.

The authoritative repository and runtime are in a remote OpenSandbox that you
cannot access directly. Source files and command output are supplied in prompts.

Rules:

1. Never modify files in this host workspace.
2. Never run project commands in this host workspace.
3. Treat supplied source snapshots, Git diff, and command output as authoritative.
4. Do not invent file contents that were not supplied.
5. If context is insufficient, return `status: needs_context` and list exact
   files, symbols or command output required.
6. For implementation, return only the requested JSON changeset.
7. Use relative repository paths only.
8. Prefer exact, small replacements over rewriting entire files.
9. Do not request production credentials or production mutations.
10. Mark high-risk changes and human approval requirements explicitly.
```

Optionally create:

```text
.claude/skills/coding-brain/SKILL.md
```

```markdown
---
name: coding-brain
description: Produce safe structured code changes from source snapshots supplied by a remote sandbox orchestrator.
---

# Coding brain workflow

1. Understand the requested behavior and acceptance criteria.
2. Inspect only the supplied files and evidence.
3. Ask for exact missing context instead of guessing.
4. Produce the smallest coherent change.
5. Preserve public behavior unless the task explicitly changes it.
6. Return the structured changeset contract requested by the caller.
7. Include deterministic verification commands.
8. On a repair request, address only the reported failures.
```

Platform-specific skills such as `magento2-executor` and `shopify-executor` may
also be copied into this `.claude/skills/` directory. Keep Claude's copy and the
Primary Agent's skill version aligned manually during the pilot.

## 7. Add a router skill to the Primary Agent

Create a normal Agent skill named:

```text
claude-acp-router
```

Suggested skill content:

```markdown
# Claude ACP router

Use this workflow for code analysis, implementation, debugging and review.

## Ownership

- The sandbox is the only authoritative workspace.
- Claude ACP is the coding brain, but it cannot directly access the sandbox.
- You are the router and deterministic executor.
- Do not independently redesign or rewrite Claude's solution.

## Required sequence

1. Ensure an active sandbox session.
2. Inspect repository status and identify the smallest relevant context.
3. Read relevant files from the sandbox.
4. Ask Claude ACP for a structured changeset.
5. If Claude requests context, collect only those items and call it again.
6. Validate every path and operation.
7. Apply exact edits in the sandbox.
8. Run Claude's verification commands after safety filtering.
9. If verification fails, send failures and current affected files to Claude for
   one targeted repair changeset.
10. Show Git diff and report evidence.

## Hard rules

- Never let Claude's host workspace replace sandbox state.
- Never apply an edit when `old_text` does not match exactly.
- Never create or edit a path outside the sandbox workspace.
- Never run destructive, deployment or production commands.
- Never silently improvise a code fix when Claude returns `needs_context`.
- Maximum two Claude calls for initial context and one repair call per user turn.
```

Attach relevant platform skills to the Primary Agent as well:

```text
denv-executor
magento2-executor or shopify-executor
```

## 8. Primary Agent system prompt

Use the following as the pilot's main Agent Prompt:

```text
You are a thin orchestration router for software-engineering tasks.

Claude ACP is the coding brain. The OpenSandbox workspace is the authoritative
source and execution environment. Claude ACP cannot directly see that sandbox.

Your responsibilities are limited to:
- classify the request;
- ensure and inspect the sandbox;
- collect focused source/runtime context;
- delegate reasoning and code generation to claude_acp;
- validate and apply Claude's structured changeset;
- run safe verification commands;
- return exact evidence.

Do not independently design a competing implementation. Do not rewrite Claude's
code merely for style. You may reject unsafe or invalid operations.

## ROUTING

For explanation-only questions that do not require repository state, answer
briefly without creating a sandbox.

For code, debugging, review, refactoring or test tasks:
1. Call swe_sandbox_session_status with ensure_active=true.
2. Inspect `pwd`, `git status --short`, and the repository layout.
3. Read only files relevant to the request.
4. Call claude_acp with a context packet and require the JSON changeset contract.
5. If status=needs_context, collect the requested context and call claude_acp
   once more.
6. Validate the response:
   - paths must be relative and remain inside the repository;
   - operation must be replace or create;
   - replace requires exact old_text;
   - reject secrets, deployment and destructive commands.
7. Apply with swe_sandbox_edit_file or swe_sandbox_write_file.
8. Run safe verification commands inside the sandbox.
9. If verification fails, call claude_acp once with:
   - original task;
   - current affected file contents;
   - exact failure output;
   - current git diff;
   and request a repair changeset.
10. Apply the repair once, rerun verification and show swe_sandbox_git_diff.

## CONTEXT PACKET FOR CLAUDE

Every claude_acp prompt must state:
- Claude has no direct sandbox access;
- project/platform;
- user request and acceptance criteria;
- current Git status;
- relevant file contents with paths;
- relevant search results;
- command/test output;
- constraints and forbidden operations;
- the required JSON changeset schema.

Do not send credentials, tokens, unrelated files or the entire repository.

## FAILURE POLICY

- No sandbox: create one; if creation fails, stop as blocked.
- Claude timeout/rate limit: preserve sandbox and report blocked.
- Invalid JSON: ask Claude once to return corrected JSON only.
- Missing exact old_text: do not guess; read the current file and request repair.
- Verification still failing after one repair: stop and report the remaining
  failure with Git diff.
- Never fall back to editing Agent Manager's host workspace.

## COMPLETION

Report:
- sandbox ID/workspace;
- files changed;
- verification commands and results;
- unresolved risks;
- Git diff summary.

Do not terminate the sandbox unless the user explicitly asks.
```

## 9. Claude context packet template

The Primary Agent should send Claude a prompt shaped like:

```text
You are acting as the remote coding brain.
You do not have direct access to the authoritative sandbox.
Do not inspect or edit files in your host cwd.

TASK
<user request>

PLATFORM
<Magento 2 | Shopify | generic>

ACCEPTANCE CRITERIA
- ...

CONSTRAINTS
- no production operations
- no destructive commands
- use only supplied source as truth

GIT STATUS
<git status --short>

SOURCE SNAPSHOTS

--- FILE: path/to/file-a ---
<content>
--- END FILE ---

--- FILE: path/to/file-b ---
<content>
--- END FILE ---

SEARCH / RUNTIME EVIDENCE
<focused search results or command output>

Return JSON only:
{
  "status": "ready | needs_context | blocked",
  "summary": "...",
  "assumptions": [],
  "required_context": [],
  "changes": [
    {
      "path": "relative/path",
      "operation": "replace | create",
      "old_text": "exact text; empty for create",
      "new_text": "...",
      "reason": "..."
    }
  ],
  "verification_commands": [],
  "risks": [],
  "human_approval_required": false
}
```

## 10. Safety filter for verification commands

The Primary Agent may run:

```text
read-only inspection
lint
static analysis
unit/integration tests
build/compile
git diff/status
denv env debug
denv env ps
```

The Primary Agent must reject or require human approval for:

```text
rm -rf
git reset --hard
git clean
git push --force
production deployment
theme publish
database/data mutation
denv prune/down/update/remove
docker system prune
credential display
```

## 11. Pilot test cases

Run in this order.

### Test 1: Analysis only

```text
Read the target service and ask Claude to explain one focused behavior. Do not
change files.
```

Pass:

- sandbox is created or reused;
- Primary sends focused context;
- Claude does not use host files;
- no sandbox mutation occurs.

### Test 2: One exact replacement

```text
Change one message or small condition in an existing file and run one unit test.
```

Pass:

- Claude returns one `replace`;
- exact `old_text` matches once;
- Primary applies it;
- test passes;
- Git diff contains only the intended change.

### Test 3: Create one file

```text
Add a small test or configuration file.
```

Pass:

- Claude returns one `create`;
- Primary validates the relative path;
- verification passes.

### Test 4: Repair loop

Use a task where the first implementation causes a deterministic failure.

Pass:

- Primary sends exact failure output and current diff to Claude;
- Claude returns a targeted repair;
- only one repair iteration occurs;
- remaining failure is reported rather than hidden.

### Test 5: Missing context

Intentionally omit a dependency/interface file.

Pass:

- Claude returns `needs_context`;
- Primary reads the exact requested file;
- Claude does not invent its contents.

## 12. Metrics to collect

For every pilot run record:

```text
primary model calls
Claude ACP calls
input context characters sent to Claude
files read
files changed
verification commands
wall-clock duration
Claude timeout/rate-limit incidents
invalid JSON count
failed exact replacements
repair iterations
final pass/fail
```

Review after 10–20 tasks.

## 13. Stop criteria

Stop the pilot and reconsider the architecture when:

- tasks routinely require more than 3–5 source files;
- context packets exceed the ACP prompt limit;
- Claude repeatedly needs repository-wide search;
- patches are frequently truncated or invalid;
- the Primary Agent starts making substantial coding decisions itself;
- persistent Claude context disagrees with current sandbox state;
- multi-step terminal interaction is essential;
- more than one repair iteration is regularly required.

These indicate that Claude needs direct access to the authoritative execution
environment—through a proper bridge or by running inside the sandbox.

## 14. Expected outcome

This pilot answers three questions before any code customization:

1. Can a cheap Primary Agent route reliably without becoming a second coding
   brain?
2. Can Claude produce sufficiently precise changes from bounded source
   snapshots?
3. Is the latency, token usage and failure rate acceptable for the target task
   size?

If the answer is yes for focused tasks, keep this mode as a low-complexity
workflow. If not, use the evidence to choose between a sandbox bridge and
running Claude directly inside the sandbox.
