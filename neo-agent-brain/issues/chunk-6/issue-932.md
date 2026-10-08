---
id: 932
title: A seat refuses to publish its own harness session id
state: CLOSED
labels:
  - enhancement
  - ai
  - security
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-08T09:52:21Z'
updatedAt: '2026-10-08T14:40:05Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/932'
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
closedAt: '2026-10-08T14:40:05Z'
---
# A seat refuses to publish its own harness session id

## Context

Operator, 2026-10-08: harness session ids (Claude Code, Codex, Desktop) must not reach future public artifacts. Memory Core mints its own session ids for consistency and security. Existing artifacts don't matter; future ones do.

Measured on one seat the same day: the full harness ids of 7 of its 14 most recent Claude sessions appear in 54 public issues and PRs. Most arrived as `Origin Session ID` lines, from sessions that passed the harness id as `add_memory`'s `sessionId`. One more leaked within the hour, through an A2A receipt a peer quoted on Brain #768 (since edited).

The rule already existed as a practice: a June seat note records "harness ids are not Memory Core ids; never override `sessionId` with one". It decayed anyway. So the fix is a guard, not another reminder.

## The Problem

Only the seat knows its own harness id at the moment it publishes. Body lints and the review validators cannot tell a harness UUID from a Memory Core UUID, so a pattern check would either block legitimate stamps or miss the leak. The check belongs where the id is known: the seat's own pre-tool hook.

## The Architectural Reality

- Fleet projects seat hooks per family from `ai/scripts/lifecycle/hooks/<family>/` through `projectSeatHooks.mjs`. Today the Claude events in `claude/events.manifest.json` are `SessionStart`, `UserPromptSubmit`, `PostToolUse` and `Stop`; there is no `PreToolUse`.
- A Claude Code hook payload carries `session_id`. A `PreToolUse` payload also carries `tool_name` and `tool_input`. The precedent for a pre-tool guard is the neo repo's tracked `.claude/hooks/rgReplaceGuardHook.mjs`, which fails open on a malformed payload.
- Codex hooks (`codex/hooks.json`) also receive `session_id`, which `codex-lane-state-stop.mjs` reads. Whether Codex exposes a pre-tool event is unmeasured. Kimi hooks are `[[hooks]]` blocks in the seat `config.toml`; the same question applies.
- The leak paths are a publishing tool's input (GitHub MCP tools, `gh` through Bash, A2A `add_message`, `add_memory`'s `sessionId`) and a written file later used as a body (`Write`/`Edit`, then `--body-file`).

## The Fix

1. Add a new Claude seat hook, `harnessIdGuardHook.mjs`, in `ai/scripts/lifecycle/hooks/claude/`. Register it as `PreToolUse` in `events.manifest.json`, with a matcher for the publishing and writing tools: `Bash`, `Write`, `Edit`, `mcp__neo-mjs-github-workflow__.*` and `mcp__neo-mjs-memory-core__add_(message|memory)`.
2. The hook refuses the call when the serialized `tool_input` contains the payload's `session_id` (exact match, case-insensitive). The refusal tells the agent to stamp the id that `add_memory` echoes instead. It fails open on a malformed payload.
3. Measure the Codex and Kimi hook contracts read-only. Projecting their guard is #933, because only a seat running those harnesses can exercise it.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Claude `PreToolUse` payload: `session_id`, `tool_input` | the Claude Code hook contract, read by `harnessIdGuardHook.mjs` | reads `session_id` and serializes `tool_input`; reads no other field | a payload missing either passes | module JSDoc | unit: decision and entrypoint |
| Match boundary | `decideHarnessIdGuard` | refuses when the trimmed, lower-cased `session_id`, at least `MIN_SESSION_ID_LENGTH` (16) characters, occurs in the lower-cased serialized `tool_input` | a shorter, non-string or missing id, a missing input, or unparsable stdin passes (fail open) | function JSDoc | unit |
| Selected tools | the `PreToolUse` matcher in `claude/events.manifest.json` | `^(Bash\|Write\|Edit\|mcp__neo-mjs-github-workflow__.*\|mcp__neo-mjs-memory-core__add_(message\|memory))$`, anchored because Claude tests a matcher regex unanchored | every other tool never starts the hook | manifest `$comment` | unit: the published pattern tested unanchored, with positive and negative names |
| Refusal output | the Engine guard's form | stdout `{"decision":"block","reason":HARNESS_ID_GUARD_MESSAGE}`; the reason never echoes the id | no output means the call runs | `HARNESS_ID_GUARD_MESSAGE` JSDoc | unit: entrypoint |
| Execution budget | the manifest entry | synchronous, `timeout: 2` s, node builtins only; the hook never exits 2 and catches its own errors, so its only refusal is the printed decision | none | manifest `$comment` | unit: entrypoint |
| Projection and custody | `projectSeatHooks.mjs`; ADR 0040 §2.7 | projected into the seat's `.claude/hooks/` and appended to `PreToolUse` after the Engine's tracked `rgReplaceGuardHook` bucket, which is never restated | none | manifest `$comment` | unit: reconcile and custody; installed witness AC-5 under #571 |

Ledger added 2026-10-08 by the ticket author, per Brain #934 review 5456112430 (RA-2).

## Acceptance Criteria

- [ ] On a Claude seat, `PreToolUse` refuses a matched tool call whose input contains the seat's own `session_id`, and passes a call without it untouched. Red first.
- [ ] A malformed or empty payload passes (fails open), and unmatched tools never start the hook.
- [ ] `projectSeatHooks` projects the hook into a seat, and the seat's settings carry the `PreToolUse` entry.
- [ ] For Codex and Kimi, the pre-tool surface is measured read-only and recorded in #933, which projects their guard. (Amended 2026-10-08: a Claude seat cannot exercise their hooks live.)
- [ ] **Post-merge, installed:** in a Fleet-launched Claude seat, posting the session's own id in an A2A body is refused with the guidance message.

## Out of Scope

- Scrubbing existing artifacts (operator: not needed).
- Ids of the seat's earlier sessions. The hook only knows the current one.
- How Memory Core itself binds a harness id (D#19401) and the `healthcheck` session field (#931).

## Avoided Traps

- **A shape lint on bodies.** Memory Core ids are UUIDs too, so a pattern match blocks legitimate stamps.
- **A hook in one repository's checkout.** Seats work in several repositories, and only Fleet projection covers all of them.
- **Another broadcast reminder.** The June practice shows that discipline alone decays.

## Related

D#19401 · #931 · #768 (the same-day leak) · practice broadcast `MESSAGE:35f11ed7`

Decision Record impact: none.

Live latest-open sweep: latest 20 open Brain issues at 2026-10-08T09:51:53Z, no equivalent. Exact search for "harness session id", "session id public", "PreToolUse guard" and "Origin Session ID harness" found no equivalent.
A2A sweep: the latest 30 messages hold no claim on this scope.
MC sweep: the problem's nouns surfaced the June practice note (Grace's seat), and no prior decision against a guard.
Own-assignment sweep: open, none overlapping (#931 is the healthcheck field; #571 is the seat layout).

Origin Session ID: 58ad7fe7-062f-4a99-a92c-e5867f6ba8aa
Retrieval Hint: "harness session id public artifact guard PreToolUse seat hook"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-10-08T09:52:21Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-08T09:52:22Z @neo-opus-ada added the `enhancement` label
- 2026-10-08T09:52:23Z @neo-opus-ada added the `ai` label
- 2026-10-08T09:52:23Z @neo-opus-ada added the `security` label
- 2026-10-08T09:52:23Z @neo-opus-ada added the `agent-os` label
- 2026-10-08T11:01:29Z @neo-opus-ada cross-referenced by #933
- 2026-10-08T11:03:21Z @neo-opus-ada cross-referenced by PR #934
- 2026-10-08T11:52:44Z @neo-opus-ada referenced in commit `34cc3aa` - "fix(hooks): anchor the harness-id guard's matcher to its declared tools (#932)

Claude tests a PreToolUse matcher regex against the tool name unanchored, so the published alternation also selected names that merely contain a declared tool, such as an unrelated MCP tool containing Bash or a suffixed add_memory. The matcher is now anchored, and the spec tests the published pattern unanchored, the way Claude applies it, with both negative controls. The uppercase control now uses a synthetic id that contains letters."
- 2026-10-08T14:39:54Z @tobiu referenced in commit `aab9e2a` - "feat(hooks): a Claude seat refuses to publish its own harness session id (#932) (#934)

* feat(hooks): a Claude seat refuses to publish its own harness session id (#932)

A new Agent-OS-owned PreToolUse hook, harnessIdGuardHook, refuses a call to
Bash, Write, Edit, a GitHub workflow tool, add_message or add_memory when the
call's input carries the payload's own session_id. Memory Core ids are UUIDs
too, so the hook matches that exact id rather than a shape. It prints the
Engine guard's {decision: block} form, uses only node builtins, and passes a
malformed payload.

The projector already appends manifest entries per event and keeps the
Engine's tracked rgReplaceGuardHook entry, so the two PreToolUse entries sit
side by side. The custody test now asserts that intent (no Engine guard
restated, and every Brain PreToolUse command runs a Brain hook source) instead
of forbidding the event outright. The census grows to nine hooks.

* fix(hooks): anchor the harness-id guard's matcher to its declared tools (#932)

Claude tests a PreToolUse matcher regex against the tool name unanchored, so the published alternation also selected names that merely contain a declared tool, such as an unrelated MCP tool containing Bash or a suffixed add_memory. The matcher is now anchored, and the spec tests the published pattern unanchored, the way Claude applies it, with both negative controls. The uppercase control now uses a synthetic id that contains letters."
- 2026-10-08T14:40:06Z @tobiu closed this issue

