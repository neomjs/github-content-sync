---
id: 944
title: 'The harness-id guard refuses artifacts, not Memory Core or temp paths'
state: CLOSED
labels:
  - bug
  - ai
  - security
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-09T03:36:57Z'
updatedAt: '2026-10-09T04:10:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/944'
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
closedAt: '2026-10-09T04:10:04Z'
---
# The harness-id guard refuses artifacts, not Memory Core or temp paths

## Context

Operator, 2026-10-09: *"we have separate crypto MC session ids to post on artifacts. the hook might be outdated and aim to not pass claude or codex harness session ids into artifacts. additionally storing them into MC should definitely be allowed."* This restates the operator's D#19401 proposal of 2026-10-08: Memory Core accepts the harness id as a separate input and binds its own minted session to it.

The #932 guard (PR #934) refuses more than that on every Fleet-launched Claude seat:
1. **Memory Core writes.** Its matcher selects `add_memory` and `add_message`, so a turn memory or an A2A body that carries the session's harness id is refused.
2. **The harness's own temp folder.** Claude Code keeps a session's `scratchpad/`, `tasks/` and `images/` under `<tmp>/claude-<uid>/<project-key>/<session-id>/`, and its system prompt sends temporary files to the scratchpad. The id sits in that path, so every `Write` there, and every `Bash` call naming such a path, is refused. Reported in Vega's #571 receipt (comment 6066238923, item 4); reproduced by Grace on her seat (2026-10-08, 18:37Z) and by Ada on hers (2026-10-09, about 03:02Z).

## The Problem

The guard exists because harness ids reached 54 public issues and PRs (#932). Memory Core is not an artifact: it is the team's own store, and it mints the ids that artifacts carry. A path in the harness's temp folder is a location the harness handed the seat, not content the seat publishes. Refusing both costs every Claude seat, every session, and protects nothing.

The leak #932 measured most often did run through Memory Core, but as an alias, not as storage: `add_memory` uses a caller-supplied `sessionId` as the session key, so a seat that passes its harness id there gets it echoed back as its "Memory Core" id and stamps it. That stamp still has to pass a GitHub tool, the shell or a written file, where the guard stays.

## The Architectural Reality

- `ai/scripts/lifecycle/hooks/claude/harnessIdGuardHook.mjs:35`: `decideHarnessIdGuard` refuses when the lower-cased `JSON.stringify(payload.tool_input)` contains the lower-cased `session_id`. It never reads `tool_name`.
- `ai/scripts/lifecycle/hooks/claude/events.manifest.json`: the `PreToolUse` matcher is `^(Bash|Write|Edit|mcp__neo-mjs-github-workflow__.*|mcp__neo-mjs-memory-core__add_(message|memory))$`. `projectSeatHooks.mjs` projects hook and matcher into each seat; Ada's installed copy equals `origin/dev` plus the generated header.
- `test/playwright/unit/ai/scripts/lifecycle/hooks/harnessIdGuardHook.spec.mjs` pins the Memory Core tools as matcher positives.
- `ai/services/memory-core/MemoryService.mjs:581`: `addMemory` falls back to `SessionService.currentSessionId` only when `sessionId` is omitted.
- Owning folder per `npm run ai:structure-map -- --files --loc`: `ai/scripts/lifecycle/hooks/claude`. No new file.

Design authority: the operator's ruling quoted in Context. The hook's own summary names publication as its purpose: "a seat never publishes its own harness session id".

## The Fix

1. **Matcher:** `^(Bash|Write|Edit|mcp__neo-mjs-github-workflow__.*)$`. The manifest `$comment` says why Memory Core is not on it.
2. **Temp root:** `decideHarnessIdGuard` masks `/claude-<uid>/<project-key>/<session-id>` in `Bash`'s `command` and in `Write`/`Edit`'s `file_path` before the match. File content and every GitHub tool's input keep the full match.
3. **Refusal text:** names artifacts only, plus the remedy for the alias: omit `add_memory`'s `sessionId` and stamp the id it echoes.
4. JSDoc and spec follow.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `PreToolUse` matcher, `claude/events.manifest.json` | operator ruling, 2026-10-09 | selects `Bash`, `Write`, `Edit` and the GitHub tools; Memory Core tools never start the hook | — | manifest `$comment` | unit: pattern tested unanchored, Memory Core tools negative |
| `decideHarnessIdGuard` match | the hook's summary (publication) | masks the session temp root in `Bash.command` and `Write`/`Edit.file_path`, then matches as today | another `tool_name` or a non-object input: no mask, today's match | function JSDoc | unit |
| `HARNESS_ID_GUARD_MESSAGE` | #932 contract | names artifacts; remedy: omit `sessionId`, stamp the echo; never echoes the id | — | constant JSDoc | unit |

## Acceptance Criteria

- [ ] On a Claude seat, a `Write` into the session's scratchpad and a `Bash` call reading a file under the session temp root run. Red first against the current hook.
- [ ] `add_memory` and `add_message` never start the guard.
- [ ] A GitHub tool input, a `Bash` command, or a written file's content that carries the session id is still refused, including a GitHub body that pastes a temp-root path.
- [ ] A malformed payload still passes, and the refusal never echoes the id.
- [ ] **Post-merge, installed:** after a Fleet Start re-projects the hooks, a Claude seat writes into its scratchpad and saves an `add_memory` that names its own harness id.

## Out of Scope

- Codex and Kimi: #933, amended to this aim.
- Memory Core taking the harness id as a separate input (D#19401).
- An earlier session's id: the hook knows only the current one.

## Avoided Traps

- **Unguarding `Write`/`Edit`.** A written file becomes an artifact through `--body-file` or a commit; only its path is a location.
- **Masking GitHub inputs.** A body that pastes a temp path would publish the id inside it.
- **Hard-coding `/private/tmp`.** The mask keys on the `claude-<uid>/<project-key>/<session-id>` segments, not the OS temp root.

Residual, accepted: a `Bash` `gh … --body "…"` that pastes a temp-root path passes, because a shell command cannot be split into locations and content without a parser.
Residual, accepted: an A2A body is Memory Core storage and passes. A peer who quotes it on GitHub publishes an id that the peer's own guard cannot recognize, which is how the #768 leak of 2026-10-08 happened. Receipts therefore keep the "this session" practice.

## Related

#932 · #934 · #933 · #571 · D#19401

Decision Record impact: none.

Live latest-open sweep: latest 20 open Brain issues at 2026-10-09T03:35:50Z, no equivalent. Exact org search "harness session id": #933, #571 and unrelated Fleet epics only.
A2A sweep: latest 30 messages, all read-states, no claim on this scope.
MC sweep: "harness session id guard refuses scratchpad write add_memory blocked hook outdated", 6 results: Grace's June note (never override `add_memory`'s `sessionId` with a harness id), #932's filing, Vega's June Stop-hook session unification (superseded by the 2026-10-08 ruling), Clio's seat notes. No prior decision against this.
Own-assignment sweep: 15 open, none overlapping.

Origin Session ID: 3290d205-ee4b-4a8e-949f-fe4995197e08 (this chat's earlier Memory Core session: 7cfd8d99-6f5b-47db-9130-36468e9083a1)
Retrieval Hint: "harness-id guard artifacts only Memory Core allowed scratchpad temp root mask"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-10-09T03:36:57Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-09T03:36:59Z @neo-opus-ada added the `bug` label
- 2026-10-09T03:36:59Z @neo-opus-ada added the `ai` label
- 2026-10-09T03:36:59Z @neo-opus-ada added the `security` label
- 2026-10-09T03:36:59Z @neo-opus-ada added the `agent-os` label
- 2026-10-09T03:42:45Z @neo-opus-ada cross-referenced by PR #945
- 2026-10-09T03:43:44Z @neo-opus-ada cross-referenced by #933
- 2026-10-09T04:10:04Z @tobiu referenced in commit `0b678f4` - "fix(hooks): the harness-id guard refuses artifacts, not Memory Core or temp paths (#944) (#945)

Operator ruling 2026-10-09: artifacts carry Memory Core's own session ids, the guard keeps Claude and Codex harness ids out of artifacts, and Memory Core may store them.

- The PreToolUse matcher drops add_memory and add_message.
- The session temp root (…/claude-<uid>/<project-key>/<session-id>) is masked in Bash's command and in Write/Edit's file_path before the match, so a scratchpad write runs. File content and GitHub tool input keep the full match.
- The refusal names artifacts, plus the alias remedy: omit add_memory's sessionId and stamp the id it echoes."
- 2026-10-09T04:10:04Z @tobiu closed this issue

