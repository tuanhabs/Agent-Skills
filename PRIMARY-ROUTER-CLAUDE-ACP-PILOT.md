# Pilot: Primary Agent as Router, Claude ACP as Coding Brain

## 1. Architecture

This pilot uses four separated responsibilities:

```text
User
  ↓
Primary LangChain Agent
  ├── creates or reuses OpenSandbox
  ├── reads source and runtime evidence
  ├── sends a bounded context packet to Claude ACP
  ├── validates Claude's JSON response
  ├── applies exact changes in OpenSandbox
  └── runs verification and reports Git evidence
        ↓
Claude ACP
  ├── loads its provisioned control workspace
  ├── reasons, delegates to Claude subagents, and reviews
  └── returns a structured changeset
        ↓
OpenSandbox
  └── authoritative filesystem, Git state, and runtime
```

| Component | Responsibility |
|---|---|
| Primary Agent | Routing, sandbox lifecycle, context collection, deterministic edits, command execution, validation, reporting |
| Claude ACP | Analysis, design, code generation, review, repair, durable project learning |
| Claude Workspace Provisioner | Materialize Claude instructions, settings, agents, Skill Packs, advanced files, and manifest revision |
| OpenSandbox | Authoritative repository and execution environment |
| Git | Durable checkpoint and final evidence |

Claude's host workspace is a control workspace, not a second project checkout.
Claude must never treat host files as the sandbox repository.

## 2. Isolation model

Use these boundaries:

| Scope | Isolation |
|---|---|
| Merchant | Security and credential policy |
| Store/project | One Primary Agent and one Claude control workspace |
| Job/conversation | Unique `thread_id` |
| Claude configuration revision | Manifest revision added to the ACP session key |
| Attempt | `initial`, `context_followup`, or `repair` packet |

Current ACP key:

```text
<agent-alias>::<thread-id>::workspace-<manifest-revision>
```

Changing `CLAUDE.md`, settings, agents, skills, or advanced files creates a new
revision. The next ACP call starts a new Claude session and reloads the updated
workspace configuration.

Do not reuse one Primary Agent across unrelated stores. Use one Agent alias and
one `AI_CODE_APP_DIR` per project.

## 3. Create the Primary Agent

Create a dedicated Agent:

```text
Name: Claude ACP Router Pilot
Alias: claude_acp_router_pilot
```

Use a low-cost model with reliable tool calling:

```text
Temperature: 0
Max output: enough for tool arguments and short orchestration messages
```

The Primary Agent is not the coding brain. It may reject unsafe or malformed
Claude output, but it should not silently redesign Claude's implementation.

## 4. Enable required features

Enable:

```text
AI Code: Claude ACP agent
Sandbox Tools
Skill Packs
Claude Workspace Provisioner
```

Recommended sandbox tools:

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

Initially disable:

```text
AI Code output passthrough
sandbox termination
repository push
PR creation
deployment and production mutation
```

Output passthrough must be off because the Primary must continue after
`claude_acp` returns, validate its response, apply the changes, and verify them.

## 5. Configure Claude ACP

Suggested settings:

```text
Runtime command: npx
Command args: -y @agentclientprotocol/claude-agent-acp
Timeout: 600 seconds
Keep ACP session alive per conversation: enabled
Auto-approve: disabled initially
Anthropic API key: empty when using Claude subscription login
```

Set the AI Code App directory to the Agent's dedicated control workspace:

```text
/app/workspaces/claude_acp_router_pilot
```

Do not point it at:

- Agent Manager source;
- another project's workspace;
- an OpenSandbox repository clone;
- a shared directory used by unrelated merchants or stores.

## 6. Configure Claude Workspace Provisioner

Open:

```text
Agent → Tools → Claude Workspace Provisioner → Configure
```

The provisioner is data-driven and does not contain Magento, Shopify, denv,
skill, or subagent presets.

### General

Enable:

```text
Provision Claude files before every ACP call
```

### CLAUDE.md

Use the following starting content:

```markdown
# Role: Remote coding brain

You are the coding and review brain behind a separate orchestration agent.

The authoritative repository and runtime are in a remote OpenSandbox. You
cannot access that sandbox directly. Source snapshots, Git state, command
output, and acceptance criteria are supplied by the Primary Agent.

## Operating rules

1. Never treat this host control workspace as the project repository.
2. Never run project, denv, Docker, test, or deployment commands here.
3. Treat the latest supplied sandbox packet as the only current repository
   truth. It overrides conversation history and memory.
4. Do not invent file contents. Request exact missing context.
5. Return exactly the response contract requested by the Primary Agent.
6. Use repository-relative paths in proposed changes.
7. Prefer small exact replacements over whole-file rewrites.
8. Never request credentials or unauthorized production operations.
9. Do not edit provisioner-managed `CLAUDE.md`, `.claude/settings.json`,
   `.claude/agents/`, `.claude/skills/`, or the workspace manifest.
10. You may write only to the configured auto-memory directory when the memory
    policy permits it.
11. Delegate focused analysis or review to configured subagents when useful.
12. Installed skills describe procedures; they do not grant sandbox access.

## Memory policy

Use auto memory only for durable knowledge likely to help future jobs for this
same project.

Good memory:

- commands whose behavior was verified by current sandbox output;
- stable architecture, naming, test, build, and review conventions;
- recurring debugging insights confirmed by evidence;
- explicit user corrections and durable preferences;
- environment constraints that remain true across jobs.

Do not remember:

- credentials, tokens, cookies, customer data, or secret values;
- source snapshots, patches, Git diffs, or command logs;
- current job state, sandbox IDs, thread IDs, or temporary branches;
- unverified assumptions or conclusions from stale conversation;
- one-off failures that do not generalize.

Keep `MEMORY.md` a concise index. Put details in topic files such as
`build.md`, `architecture.md`, or `debugging.md`. Update existing entries
instead of creating duplicates.

Before relying on memory, compare it with the latest sandbox packet. If they
conflict, use the sandbox packet and correct or remove stale memory.
```

### settings.json

Use a dedicated memory directory for this Agent:

```json
{
  "autoMemoryEnabled": true,
  "autoMemoryDirectory": "/app/workspaces/claude_acp_router_pilot/.agent-manager/memory",
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)"
    ]
  }
}
```

`autoMemoryDirectory` must be absolute or start with `~/`. Never share it
between unrelated projects.

### Agents

Agents are editable `.md` files with Claude-supported YAML frontmatter.
Start with a small allowlist.

Example `researcher.md`:

```markdown
---
name: researcher
description: Analyze supplied source snapshots and request exact missing context
tools: Read, Grep, Glob
model: haiku
maxTurns: 12
memory: project
---

Analyze only the evidence supplied by the parent. Return concise findings and
exact context requests. Never inspect or edit host files as project source.
```

Example `implementer.md`:

```markdown
---
name: implementer
description: Produce a minimal structured changeset from sufficient supplied context
tools: Read, Grep, Glob
model: inherit
maxTurns: 16
memory: project
---

Produce the smallest coherent implementation. Return content through the
response contract requested by the parent. Do not edit host project files.
```

Example `reviewer.md`:

```markdown
---
name: reviewer
description: Review a proposed changeset for correctness, regressions, and security
tools: Read, Grep, Glob
model: sonnet
maxTurns: 12
memory: project
---

Review only supplied changes and evidence. Separate blocking findings from
non-blocking improvements and request exact missing evidence.
```

Avoid `isolation: worktree` in this pilot. The host control workspace is not the
authoritative Git repository.

### Skills

Select Claude skills from the same Skill Pack stores used by Primary Agents.
The provisioner copies the complete selected pack:

```text
SKILL.md
scripts/
references/
assets/
```

Typical selections:

```text
denv-executor
shopify-executor
magento2-executor
```

There are no hard-coded skill names in the provisioner. Import or sync a Skill
Pack first, then select it in the Provisioner UI.

### Advanced files

Use this section for optional Claude project files:

```text
.claude/rules/*.md
.claude/commands/*.md
.mcp.json
other project-scoped Claude configuration
```

Do not use Advanced files to overwrite `CLAUDE.md`, settings, agents, skills,
or `.agent-manager/` files. Those paths are reserved and rejected.

Save the configuration, then click:

```text
Provision now
```

The provisioner backs up unmanaged conflicts, writes files atomically, removes
disabled managed files, and records hashes in:

```text
.agent-manager/claude-workspace-manifest.json
```

## 7. Configure Primary Agent Skill Packs

Enable and attach:

```text
claude-acp-router
denv-executor, when denv applies
shopify-executor or magento2-executor, when applicable
```

For initial testing, use `full` injection for `claude-acp-router`. The Primary
must load `denv-executor` when `.denv.env`, `.denv/`, or a denv request is
present.

Claude needs its own selected copy of a Skill Pack in the Provisioner. Naming a
skill in an ACP packet does not transmit its content.

The canonical router skill is:

```text
skills/claude-acp-router/SKILL.md
```

Do not maintain another embedded router-skill copy in this guide.

## 8. Primary Agent system prompt

Use:

```text
You are a thin orchestration router for software-engineering tasks.

Claude ACP is the coding brain. OpenSandbox is the authoritative repository and
execution environment. Claude ACP cannot directly see the sandbox.

For explanation-only requests that do not require repository state, answer
briefly without creating a sandbox.

For code, debugging, review, refactoring, environment, or test tasks:
1. Ensure an active sandbox session.
2. Inspect pwd, Git status, project markers, and the smallest relevant context.
3. Load relevant Skill Packs. Load denv-executor when denv applies.
4. Collect exact source snapshots and sanitized runtime evidence.
5. Call claude_acp with the complete versioned context packet.
6. If Claude returns needs_context, apply nothing; collect only the requested
   context and call Claude once more.
7. Validate response version, task id, paths, operations, exact old_text, and
   verification commands.
8. Apply accepted edits only through swe_sandbox tools.
9. Run safe verification inside OpenSandbox.
10. On failure, send one repair packet containing current affected files, exact
    failure output, and current Git diff.
11. Rerun verification and show final Git evidence.

Do not independently create a competing implementation.
Do not use Claude host files as repository source.
Do not send credentials, unrelated files, or the entire repository.
Do not terminate the sandbox unless the user explicitly asks.
```

## 9. Primary-to-Claude context packet

Every ACP call must be self-contained. Persistent ACP conversation state and
Claude memory are helpful hints, not repository consistency boundaries.

```text
PACKET_VERSION: 1
TASK_ID: <thread-id or stable job id>
TURN_KIND: initial | context_followup | repair

ROLE
You are the remote coding brain.
You cannot access the authoritative OpenSandbox.
Do not inspect, edit, or execute host files as project source.

TASK
<verbatim user request>

PROJECT
<project/store identifier>

ACCEPTANCE_CRITERIA
- ...

ACTIVE_SKILLS
- claude-acp-router
- denv-executor
- <project skill>

CONSTRAINTS
- supplied snapshots and runtime evidence are current repository truth
- no secrets, destructive operations, or unauthorized production actions
- return the requested JSON contract only

GIT_STATUS
<git status --short>

DENV_CONTEXT
present: <true | false>
platform: <safe DENV_PLATFORM value or unknown>
preflight: <sanitized denv and Docker checks>
rendered_services: <focused denv env debug or denv env ps output>
allowed_commands: denv env debug, denv env ps, safe tests and builds
forbidden_commands: denv prune/down/update/remove, docker system prune

SOURCE_SNAPSHOTS
--- FILE: relative/path/to/file-a ---
<exact content>
--- END FILE ---

RUNTIME_EVIDENCE
--- COMMAND: exact command ---
<sanitized stdout/stderr>
--- END COMMAND ---

RESPONSE_CONTRACT
Return one JSON object with no Markdown fence.
```

Omit `DENV_CONTEXT` only when denv is irrelevant. Never send:

- credentials or authentication state;
- `.env` secret values;
- `~/.denv/auth` or `~/.denv/secrets`;
- unrelated files;
- unbounded logs;
- the entire repository.

## 10. Claude-to-Primary response contract

Require:

```json
{
  "contract_version": 1,
  "task_id": "same TASK_ID received",
  "status": "ready | needs_context | blocked",
  "summary": "brief summary",
  "assumptions": [],
  "required_context": [
    {
      "kind": "file | search | command",
      "value": "exact relative path, query, or safe command",
      "reason": "why it is required"
    }
  ],
  "changes": [
    {
      "path": "relative/path",
      "operation": "replace | create",
      "old_text": "exact existing text; empty for create",
      "new_text": "replacement or complete new file",
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

Handling:

| Status | Primary action |
|---|---|
| `ready` | Validate, apply exact changes, run approved verification |
| `needs_context` | Apply nothing; collect only requested context and resend a complete packet |
| `blocked` | Apply nothing; preserve sandbox and report blocker |

Reject the response when:

- JSON is invalid;
- `contract_version` is unsupported;
- `task_id` differs;
- a path is absolute or escapes the repository;
- an operation is not `replace` or `create`;
- `replace.old_text` is absent or does not match exactly once;
- a command is interactive, destructive, credential-reading, or production-facing;
- Claude treats host state or memory as newer than sandbox evidence.

Ask once for corrected JSON when formatting is invalid. Never guess missing
source or modify `old_text` to force a match.

## 11. Verification safety policy

Allowed by default:

```text
read-only inspection
lint and formatting checks
static analysis
unit and integration tests
build and compile
git status and diff
denv env debug
denv env ps
```

Reject or require explicit human approval:

```text
rm -rf
git reset --hard
git clean
git push --force
production deployment
Shopify live-theme publish
database or customer-data mutation
denv prune/down/update/remove
docker system prune
credential display
```

The Primary executes all denv, Docker, test, build, and Git commands inside
OpenSandbox. Claude only proposes them.

## 12. Memory consistency

Claude auto memory may improve future reasoning, but it is not authoritative.

Order of truth:

```text
latest OpenSandbox evidence
  > current context packet
  > project instructions and skills
  > Claude auto memory
  > earlier ACP conversation
```

Memory must remain project-scoped. Use a different `autoMemoryDirectory` for
each Primary Agent/store.

The Primary should periodically audit:

```text
<autoMemoryDirectory>/MEMORY.md
<autoMemoryDirectory>/*.md
```

If higher assurance is needed, extend the Claude response contract with
`context_updates` and require Primary/human approval before persistence.

## 13. Test procedure

### Test 0: Provisioning

1. Save Provisioner configuration.
2. Click `Provision now`.
3. Inspect:

```bash
find /app/workspaces/claude_acp_router_pilot -maxdepth 5 -type f
cat /app/workspaces/claude_acp_router_pilot/.agent-manager/claude-workspace-manifest.json
```

Pass:

- configured files exist;
- only selected skills and enabled agents exist;
- manifest contains hashes and a revision;
- changing configuration changes the revision.

### Test 1: Analysis only

```text
Create or reuse a sandbox. Read one relevant file and ask Claude ACP to explain
one focused behavior. Do not modify files.
```

Pass:

- Primary sends focused evidence;
- Claude does not use host files as project source;
- no sandbox mutation occurs.

### Test 2: Exact replacement

```text
Delegate a small existing-file change to Claude ACP, apply its JSON changeset,
run one targeted test, and show Git diff.
```

Pass:

- Claude returns one exact `replace`;
- `old_text` matches once;
- verification passes;
- Git diff contains only intended changes.

### Test 3: File creation

```text
Delegate creation of one small test or source file and verify it.
```

Pass:

- Claude returns `create`;
- Primary validates the relative path;
- the file is created in OpenSandbox, not the Claude host workspace.

### Test 4: Missing context

Omit a required dependency file.

Pass:

- Claude returns `needs_context`;
- Primary applies nothing;
- Primary reads only the requested context and resends a full packet.

### Test 5: Repair loop

Use a deterministic failing case.

Pass:

- Primary sends exact failure output, current affected files, and Git diff;
- Claude returns one targeted repair;
- only one repair iteration occurs;
- remaining failure is reported rather than hidden.

### Test 6: Memory

Provide a verified durable project convention and ask Claude to remember it.

Pass:

- Claude writes only under `autoMemoryDirectory`;
- it does not copy source, patch, logs, job IDs, or secrets;
- a later thread recalls the convention;
- current contradictory sandbox evidence overrides and corrects stale memory.

### Test 7: Configuration revision

Edit an agent or selected skill and save/provision again.

Pass:

- manifest revision changes;
- next ACP call uses a fresh session key;
- disabled managed files are removed;
- modified stale files are backed up before removal.

## 14. Metrics

Record:

```text
primary model calls
Claude ACP calls
context packet characters
files read and changed
verification commands
wall-clock duration
invalid JSON count
failed exact replacements
needs_context rounds
repair rounds
memory reads/writes
manifest revision
final pass/fail
```

Review after 10–20 representative tasks.

## 15. Stop criteria

Stop or redesign the bridge when:

- tasks routinely require broad repository search;
- context packets exceed practical limits;
- patches are frequently truncated or invalid;
- Primary starts making substantial coding decisions;
- Claude memory repeatedly conflicts with sandbox state;
- multi-step terminal interaction is essential;
- more than one repair iteration is regularly required;
- host Git worktrees become necessary for Claude to operate.

These indicate Claude needs direct access to the authoritative execution
environment through a native bridge or by running inside OpenSandbox.

## 16. Worktree policy

Do not enable Claude subagent `isolation: worktree` in this pilot.

Current state:

```text
OpenSandbox repository = authoritative source
Claude host workspace  = instructions, skills, agents, and memory
```

A host Git worktree would create a second writable source of truth. Introduce
worktrees only after implementing either:

1. a shared repository mount visible consistently to Claude and OpenSandbox; or
2. an explicit commit/patch synchronization protocol with conflict handling.

## 17. Expected outcome

The pilot should answer:

1. Can a cheap Primary Agent route reliably without becoming a second coding
   brain?
2. Can Claude generate precise changes from bounded sandbox evidence?
3. Do provisioned skills and subagents improve quality without mixing projects?
4. Is project-scoped memory useful without becoming stale authority?
5. Are latency, token usage, and failure rate acceptable?

If yes, retain this as a focused-task workflow. If not, use the collected
evidence to choose a native sandbox bridge or Claude execution inside the
sandbox.

## Official references

- [Claude Code subagents](https://code.claude.com/docs/en/sub-agents)
- [Claude Code memory](https://code.claude.com/docs/en/memory)
- [Claude Code settings](https://code.claude.com/docs/en/settings)
- [Claude Code worktrees](https://code.claude.com/docs/en/worktrees)
