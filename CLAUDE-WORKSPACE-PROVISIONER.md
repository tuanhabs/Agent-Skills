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
│   ├── backups/
│   └── memory/                       # optional autoMemoryDirectory
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

Use this as the starting content in the provisioner's `CLAUDE.md` field:

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
7. Do not edit `CLAUDE.md`, `.claude/settings.json`, `.claude/agents/`,
   `.claude/skills/`, or `.agent-manager/claude-workspace-manifest.json`.
8. You may write only to the configured auto-memory directory when the memory
   policy below permits it.

## Memory policy

Use auto memory only for durable knowledge likely to help future jobs for this
same project.

Good memory:

- commands whose behavior was verified by current sandbox output;
- stable architecture, naming, test, build, or review conventions;
- recurring debugging insights confirmed by evidence;
- explicit user corrections and durable preferences;
- known environment constraints that remain true across jobs.

Do not remember:

- credentials, tokens, cookies, customer data, or secret values;
- complete source snapshots, generated patches, Git diffs, or command logs;
- current job state, temporary branch names, sandbox IDs, or thread IDs;
- unverified assumptions or conclusions based only on stale conversation;
- one-off failures that do not generalize.

Keep `MEMORY.md` a concise index. Put detailed durable notes in topic files
such as `build.md`, `architecture.md`, or `debugging.md`. Update an existing
entry instead of creating duplicates.

Before relying on memory, compare it with the latest sandbox packet. If they
conflict, use the sandbox packet and correct or remove the stale memory.
```

Configure a dedicated per-Agent memory directory in the provisioner's
`settings.json` field:

```json
{
  "autoMemoryEnabled": true,
  "autoMemoryDirectory": "/app/workspaces/<agent-alias>/.agent-manager/memory"
}
```

`autoMemoryDirectory` must be absolute or begin with `~/`. Give every
project/Primary Agent a different directory so Merchant or Store learnings do
not mix. Claude Code creates and maintains `MEMORY.md` and topic files there.

Project facts that must always be present belong in `CLAUDE.md`. Repeatable
procedures belong in skills. Specialist roles belong in `.claude/agents/`.
Memory location and policy are configured through `settings.json`; the
provisioner does not impose a memory directory.

The permission above applies only to auto memory. Generated instructions,
agents, skills, settings, and manifests remain controlled by the provisioner
UI. For high-assurance workflows, extend the Claude response contract with
reviewable `context_updates` and let the Primary approve them before memory is
changed.

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
