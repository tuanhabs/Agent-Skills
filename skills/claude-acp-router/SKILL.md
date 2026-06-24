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

## Primary-to-Claude data contract

For every `claude_acp` call, build one self-contained context packet from the
current sandbox state. Never assume Claude remembers an earlier packet.

The packet must contain:

- `PACKET_VERSION`, stable `TASK_ID`, and `TURN_KIND`;
- verbatim task, platform, acceptance criteria, and constraints;
- active skills, including `denv-executor` when denv applies;
- current Git status;
- exact source snapshots with relative paths and delimiters;
- sanitized search, runtime, test, and denv evidence;
- the JSON response schema below.

When denv applies, include:

```text
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
  "changes": [
    {
      "path": "relative/path",
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

- `ready`: validate and apply changes, then run approved verification.
- `needs_context`: apply nothing; collect only the requested context and resend
  a complete packet with `TURN_KIND: context_followup`.
- `blocked`: apply nothing; report the blocker with sandbox evidence.
- Invalid JSON, mismatched `task_id`, unsafe paths/commands, unknown operations,
  or missing exact `old_text`: reject and request one correction.

## denv coordination

Load and follow `denv-executor` whenever `.denv.env`, `.denv/`, or a denv
request is present. Claude should have the same skill installed in its control
workspace; naming a skill in the packet does not transmit its instructions.

Claude proposes commands, but the Primary Agent owns execution:

1. Run denv/Docker preflight in the sandbox.
2. Sanitize results before sending `DENV_CONTEXT`.
3. Reject interactive, destructive, credential-reading, update, and production
   commands unless explicitly authorized.
4. Run `denv env debug` before `denv env up`.
5. Treat sandbox output as authoritative over Claude's assumptions.

## Required sequence

1. Ensure an active sandbox session.
2. Inspect Git status, platform markers, and relevant skills.
3. If denv applies, load `denv-executor` and run its safe preflight.
4. Read the smallest relevant source and runtime context from the sandbox.
5. Send Claude a complete context packet and response contract.
6. If Claude requests context, collect only those items and call it again.
7. Validate every response field, path, operation, and command.
8. Apply exact edits in the sandbox.
9. Run Claude's verification commands after safety filtering.
10. If verification fails, send failure output, affected files, and Git diff in
    one `repair` packet.
11. Show Git diff and report evidence.

## Hard rules

- Never let Claude's host workspace replace sandbox state.
- Never apply an edit when `old_text` does not match exactly.
- Never create or edit a path outside the sandbox workspace.
- Never run destructive, deployment or production commands.
- Never silently improvise a code fix when Claude returns `needs_context`.
- Maximum two Claude calls for initial context and one repair call per user turn.
