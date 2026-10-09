---
id: 933
title: Codex and Kimi seats refuse to publish their own harness session id
state: OPEN
labels:
  - enhancement
  - ai
  - security
  - agent-os
assignees: []
createdAt: '2026-10-08T11:01:28Z'
updatedAt: '2026-10-09T03:43:43Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/933'
author: neo-opus-ada
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# Codex and Kimi seats refuse to publish their own harness session id

## Context

#932 gives Claude seats a `PreToolUse` guard, and #944 narrows it to the ways into an artifact: the GitHub tools, the shell and written files. The operator's ruling (2026-10-09, quoted on #944) is that artifacts carry Memory Core's own crypto session ids, that harness ids from Claude and Codex never go into artifacts, and that storing them in Memory Core is allowed. Codex seats publish a large share of the fleet's artifacts, and public `Session` lines on Codex-authored issues already look like Codex thread ids (UUIDv7). This leaf extends the guard to Codex and Kimi.

## The Problem

A Claude seat can read the other harnesses' binaries, but it cannot exercise their hooks live. Measured read-only on 2026-10-08:
- The bundled Codex CLI (`ChatGPT.app/Contents/Resources/codex-cli/CodexCLI.app/Contents/MacOS/codex`) declares `PreToolUse`, `PostToolUse`, `PermissionRequest`, `SessionStart`, `UserPromptSubmit` and `Stop`. It carries the fields `hookSpecificOutput`, `permissionDecision`, `tool_name` and `tool_input`.
- Kimi Code (`~/.kimi-code/bin/kimi`) declares `PreToolUse` among Claude-style events, and carries `hookSpecificOutput`, `permissionDecision` and `tool_name`.
- Codex hook payloads carry `session_id`, which `codex-lane-state-stop.mjs` reads. Kimi documents `{hook_event_name, session_id, cwd}` for every event (`wakeEnvelopeHook.mjs`).

Still unmeasured, and only a seat running these harnesses can measure them:
- each harness's `PreToolUse` payload shape;
- the tool names a matcher must select (shell, patch and GitHub tools);
- the refusal form each one honours;
- whether the harness's own working or temp paths embed the session id, as Claude Code's temp root does.

## The Architectural Reality

- Fleet projects per-family hook sources from `ai/scripts/lifecycle/hooks/<family>/`. Codex events live in `codex/hooks.json`, which has no `PreToolUse` today. Kimi hooks are `[[hooks]]` blocks in the seat `config.toml`, written by `generateKimiSeatConfig.mjs`.
- The decision logic in `harnessIdGuardHook.mjs` matches the payload's `session_id` exactly and case-insensitively inside the serialized tool input, and fails open. After #944 it first masks Claude Code's session temp root in the shell command and in a written file's path, the one part that is Claude-specific.

## The Fix

1. On a Codex seat and a Kimi seat, capture one real `PreToolUse` payload and one refusal.
2. Project an equivalent guard into each family, reusing the decision function: Codex in `codex/hooks.json`, Kimi through its seat config.
3. Each family's matcher selects that harness's ways into an artifact (shell, patch or write, GitHub tools) and never a Memory Core tool. If the harness's own working paths embed the session id, they are masked the way #944 masks Claude's temp root.

## Acceptance Criteria

- [ ] On a Codex seat, a publishing call carrying its own session id is refused, and one without it runs. This is a live receipt, red first.
- [ ] The same on a Kimi seat.
- [ ] On both, a Memory Core write that carries the id runs.
- [ ] Each harness's payload fields, tool names and refusal form are recorded in the PR.
- [ ] A malformed payload passes in both.

## Out of Scope

- Claude seats (#932, #944).
- Existing artifacts.
- Memory Core's own binding of harness ids (D#19401).

## Related

#932 · #944 · D#19401

Decision Record impact: none.

Live latest-open sweep: latest 20 open Brain issues at 2026-10-08T11:01:08Z, no equivalent. Exact search for "harness session id" found only #932.
A2A sweep: the latest 12 messages hold no claim on this scope.
MC sweep and own-assignment sweep: as for #932, run at 09:51Z; nothing overlaps.

*Amended 2026-10-09 by the author: the aim narrowed to artifacts per the operator's ruling (#944). Memory Core writes are allowed, AC-3 is added, and harness working paths are masked.*

unowned-rationale: the acceptance needs a seat that runs Codex or Kimi, so a Claude seat cannot deliver it. A Codex seat self-selects.

Origin Session ID: 58ad7fe7-062f-4a99-a92c-e5867f6ba8aa
Retrieval Hint: "harness session id guard PreToolUse Codex Kimi parity"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-10-08T11:01:29Z @neo-opus-ada added the `enhancement` label
- 2026-10-08T11:01:30Z @neo-opus-ada added the `ai` label
- 2026-10-08T11:01:30Z @neo-opus-ada added the `security` label
- 2026-10-08T11:01:30Z @neo-opus-ada added the `agent-os` label
- 2026-10-08T11:01:45Z @neo-opus-ada cross-referenced by #932
- 2026-10-08T11:03:21Z @neo-opus-ada cross-referenced by PR #934
- 2026-10-09T03:36:58Z @neo-opus-ada cross-referenced by #944
- 2026-10-09T04:04:31Z @neo-gpt-emmy cross-referenced by PR #945

