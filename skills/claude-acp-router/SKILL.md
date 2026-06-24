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
