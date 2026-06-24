# Claude workspace provisioner

## Goal

Automatically prepare the Claude ACP control workspace for each Primary Agent.
The plugin removes the manual steps required to create `CLAUDE.md`, Claude
skills, subagents, settings, and memory directories.

This does not connect Claude directly to OpenSandbox. The sandbox remains the
authoritative repository and execution environment. Claude receives source and
runtime evidence through the `claude-acp-router` context packet.

## MVP architecture

```text
Primary Agent
  ├── authoritative code and commands: OpenSandbox
  └── claude_acp
        ├── resolve AI_CODE_APP_DIR
        ├── call claude_workspace_provision service
        ├── create/update generated Claude files
        ├── add manifest revision to the ACP session key
        └── start/resume Claude ACP in the provisioned cwd
```

Default workspace:

```text
/app/workspaces/<agent-alias>/
├── CLAUDE.md
├── .agent-manager/
│   ├── claude-workspace-manifest.json
│   └── backups/
├── .claude/
│   ├── settings.json
│   ├── skills/<selected-skill-pack>/
│   ├── agents/<configured-agent>.md
│   ├── rules/<optional-rule>.md
│   └── commands/<optional-command>.md
└── <other configured project files>
```

Only files listed in the manifest are managed by the plugin. Job notes and
other user-created files are preserved.

## Data-driven configuration

The provisioner contains no platform, skill, or subagent presets. Its UI edits
five data sections:

1. `CLAUDE.md`: stable project instructions written verbatim.
2. `settings.json`: a validated JSON object.
3. Agents: CRUD entries with `filename`, `content`, and `enabled`.
4. Skills: selections discovered dynamically from the same shared and
   Git-backed Skill Pack stores used by Primary Agents.
5. Advanced files: optional path/content entries for Claude rules, commands,
   MCP configuration, hooks, or future Claude project features.

Each selected Skill Pack is copied completely, including `SKILL.md`,
`scripts/`, `references/`, and `assets/`. Adding a new commerce or project
skill requires importing a Skill Pack, not changing provisioner code.

## Isolation model

Use these boundaries:

| Scope | Boundary |
|---|---|
| Merchant | Security and credential policy |
| Store/project | One Primary Agent and one Claude control workspace |
| Job | Unique `thread_id` and context packet |
| Attempt | `TURN_KIND=initial`, `context_followup`, or `repair` |
| Claude subagent | Fresh specialized context inside the Claude session |

For the MVP, Agent alias is the project identity:

```text
workspace = /app/workspaces/<agent-alias>
ACP key   = <agent-alias>::<thread-id>::<manifest-revision>
```

Later, Coding Agent can map this explicitly:

```text
merchant-<id>/store-<id>/jobs/job-<id>
```

## Provision lifecycle

Provision immediately before each Claude ACP call:

1. Resolve the effective ACP working directory.
2. Ask enabled plugins for `claude_workspace_provision`.
3. Validate and materialize the saved Claude.md, settings, agents, skills, and
   advanced files.
4. Write generated files atomically.
5. Store file hashes and template version in the manifest.
6. Return the workspace path and manifest revision.
7. Include the revision in the ACP session key.

Provisioning is idempotent. An unchanged template produces the same revision
and reuses the existing ACP session. A changed template produces a new revision
and therefore starts a new session that reloads Claude instructions.

## Claude instructions and memory

`CLAUDE.md` contains stable behavior only:

- Claude is a remote coding brain.
- OpenSandbox state supplied by the Primary is authoritative.
- Claude must not inspect or mutate its host workspace as if it were the repo.
- Claude must return the response contract supplied in the request.
- Claude may delegate to the provisioned subagents.

Project facts that must always be present belong in `CLAUDE.md`. Repeatable
procedures belong in skills. Specialist roles belong in `.claude/agents/`.
Memory location and policy are configured through `settings.json`; the
provisioner does not impose a memory directory.

Do not let Claude rewrite generated instructions directly. Proposed durable
context should eventually be returned as structured `context_updates` and
approved by the Primary Agent before persistence.

## Subagents

Subagents are normal editable Claude Markdown files. Administrators can add,
edit, disable, or delete them without deploying code. Each file should contain
Claude-supported YAML frontmatter and a system prompt body.

Subagent files are loaded when a Claude session starts. Changing them requires
a fresh session; the manifest revision in the session key provides this.

## Why worktrees are disabled in the MVP

Claude's host workspace is currently a control workspace, while OpenSandbox is
the repository of record. Creating a Git worktree on the host would create a
second writable repository and a split-brain risk:

```text
OpenSandbox Git state != Claude host worktree Git state
```

Use Claude worktrees only after implementing one of:

1. a shared repository mount visible to both Claude and OpenSandbox; or
2. a commit/patch synchronization protocol with explicit conflict handling.

Until then, isolate jobs with `thread_id`, context packets, and job directories,
not host Git worktrees.

## Test procedure

1. Enable the `Claude Workspace Provisioner` tool group for a test Agent.
2. Open its configuration and enable provisioning.
3. Configure `CLAUDE.md` and `settings.json`.
4. Add the required subagents and select Skill Packs.
5. Ensure AI Code `App directory` points to the Agent workspace.
6. Start a new Playground thread.
7. Ask the Agent to delegate a small file change through Claude ACP.
8. Inspect the host workspace:

```bash
find /app/workspaces/<agent-alias> -maxdepth 4 -type f -print
cat /app/workspaces/<agent-alias>/.agent-manager/claude-workspace-manifest.json
```

9. Edit a provisioner setting and call Claude ACP again. The manifest revision
   should change and a fresh ACP session should be created.

## Next phase

After the MVP is stable:

- bind workspace identity to `CodingStore`, not only Agent alias;
- add a job workspace and explicit job/session mapping;
- add structured, reviewable memory updates;
- expose provision status and manifest drift in the UI;
- add cleanup/retention policies;
- evaluate shared-repository or patch-sync worktrees.

## Official references

- [Claude Code subagents](https://code.claude.com/docs/en/sub-agents)
- [Claude Code memory](https://code.claude.com/docs/en/memory)
- [Claude Code settings](https://code.claude.com/docs/en/settings)
- [Claude Code worktrees](https://code.claude.com/docs/en/worktrees)
