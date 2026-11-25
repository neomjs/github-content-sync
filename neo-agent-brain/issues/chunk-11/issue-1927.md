---
id: 1927
title: Verify live post-reload get_message read stamp and broadcast drain (#1900 AC-6)
state: CLOSED
labels:
  - enhancement
  - ai
  - testing
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-25T14:34:03Z'
updatedAt: '2026-11-25T20:01:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1927'
author: neo-gemini-julia
commentsCount: 2
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1900 Reading a message marks it read; a session end drains its broadcasts'
blocking: []
closedAt: '2026-11-25T20:01:32Z'
---
# Verify live post-reload get_message read stamp and broadcast drain (#1900 AC-6)

## Context

PR #1925 (`Resolves #1900`) implements recipient auto-read stamping at the Memory Core MCP `get_message` adapter boundary (`readReceipt` + `keepUnread: true` opt-out) and adds `mark_read({all: true, includeUnseenBroadcasts: true})` for session-end broadcast drains while rejecting `includeUnseen: true` + `includeUnseenBroadcasts: true` combinations.

Per #1900 AC-6 (`[post-merge] One seat's session: its get_message calls with no mark_read by id, and the receipt of its session-end drain, posted here`) and Round-1 review `RA-3` on PR #1925 (`PRR_kwDOUBzDFM8AAAABaSXXXg`), #1555 owns `projection: 'counts'` size and explicitly excludes unseen-broadcast drains, so AC-6 requires a dedicated, owned post-reload verification record linked back to #1900.

Curated labels (`good first issue`, `help wanted`) are omitted because verifying AC-6 requires an active maintainer seat session against the live reloaded Memory Core plane.

## The Problem

Unit tests (`MailboxService.spec.mjs`) verify `get_message` auto-stamping, degraded `readReceipt` outcomes, and `includeUnseenBroadcasts` SQL selection in CI, but live-plane validation requires Memory Core (`127.0.0.1:3102`) to reload onto merged `dev` first, followed by filing the `neo-agent-skills` sunset/recovery rule update noted in #1900's Out of Scope.

## The Architectural Reality

- `ai/services/memory-core/MailboxService.mjs` (`getMessage`, `markRead`, `_markUnreadSnapshotRead`) and `ai/mcp/server/memory-core/toolService.mjs` (`getMessageTool`).
- `ai/mcp/server/memory-core/openapi.yaml` (`get_message`, `mark_read`).
- #1900 Out of Scope defers updating `.agents/skills/session-sunset/references/session-sunset-workflow.md` Step 7, `.agents/skills/context-recovery/references/context-recovery-workflow.md`, and `AGENTS.md` until the Brain surface lands on `dev`.

## The Fix

Once PR #1925 merges to `dev` and the live Memory Core plane reloads onto a head containing #1925:
1. Execute one active seat session's `get_message` calls (verifying `readAt` + `readReceipt: {status: 'read', durable: true}` with zero per-id `mark_read` follow-ups) and run `mark_read({all: true, includeUnseenBroadcasts: true})` at session end, verifying unseen direct messages remain withheld (`withheldUnseenCount`).
2. Post the exact live-plane tool receipts on both this ticket and #1900.
3. File (or ship) the `neomjs/neo-agent-skills` follow-up updating `session-sunset` Step 7 and `context-recovery` mailbox guidance to use automatic `get_message` stamping and `mark_read({all: true, includeUnseenBroadcasts: true})`.

Decision Record impact: aligned-with `adr:0002-phase3-wake-substrate-standards-alignment` (as amended by #1925).

## Acceptance Criteria

- [ ] Verify the running Memory Core plane (`inspect_deployment` / `healthcheck`) includes merged PR #1925.
- [ ] Record a live seat session's `get_message` auto-read receipt (with no per-id `mark_read`) and a `mark_read({all: true, includeUnseenBroadcasts: true})` receipt preserving unseen direct messages, and link the receipt back on #1900.
- [ ] File the `neomjs/neo-agent-skills` runbook follow-up noted in #1900 Out of Scope and link it here before closing.

## Out of Scope

- `list_messages({projection: 'counts'})` payload capping (#1555) and archive decay (#87).
- Changes to `MailboxService.mjs` code already delivered in PR #1925 unless live reload surfaces a regression.

## Related

- #1900 (parent feature ticket; blocks this post-merge verification)
- #1925 (delivering PR)
- #1555 (sibling `counts` size ticket)

owned: self-assigned to `@neo-gemini-julia` (`blocked_by` #1900).

Live latest-open + closed sweep: checked latest 20 open (#1920..#1735) + open and closed search (`get_message mark_read includeUnseenBroadcasts 1900`; Out of Scope / Avoided Traps read on #1900 and #1555) at 2026-11-25T14:32Z; no equivalent post-reload AC-6 owner found. 155 open (red): #1900 closes on PR #1925 merge and #1555 excludes unseen-broadcast drains, requiring this bounded post-merge owner per PR #1925 RA-3.
A2A in-flight claim sweep: `list_messages({status: 'all', projection: 'compact', limit: 20})` at 2026-11-25T14:32Z; no competing claim.
MC sweep: `query_raw_memories` on `get_message includeUnseenBroadcasts 1900 1925` (5 hits); no duplicate ticket.
Own-assignment sweep: 3 open (#1919, #1900, #1230), none covering post-reload verification.
Structure map: `ai` root (`93` folders, `ai/services/memory-core/MailboxService.mjs`); no new `.mjs` file.

Origin Session ID: 05580ab8-2f98-4b02-891c-75d444d212ec
Retrieval Hint: "post-merge live receipt get_message readReceipt includeUnseenBroadcasts 1900"

## Timeline

- 2026-11-25T14:34:04Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-25T14:34:06Z @neo-gemini-julia added the `enhancement` label
- 2026-11-25T14:34:06Z @neo-gemini-julia added the `ai` label
- 2026-11-25T14:34:07Z @neo-gemini-julia added the `testing` label
- 2026-11-25T14:34:07Z @neo-gemini-julia added the `agent-os` label
- 2026-11-25T14:36:30Z @neo-gemini-julia marked this issue as being blocked by #1900
- 2026-11-25T14:36:33Z @neo-gemini-julia cross-referenced by #1900
- 2026-11-25T14:45:16Z @neo-gemini-julia cross-referenced by PR #1925
### @neo-gemini-julia - 2026-11-25T15:20:02Z

Unblocked by merged PR #1925 (`63e19a3f`). Checked live Memory Core (`inspect_deployment` + `get_message` responses in session `05580ab8-2f98-4b02-891c-75d444d212ec` at `2026-11-25T15:14Z`): the running MCP plane (`127.0.0.1:3102`) is still on its pre-`63e19a3f` runtime (no `readReceipt` returned yet on `get_message`). Keeping self-assigned to capture the live `AC-1`/`AC-2` post-reload receipts and `AC-3` skills runbook follow-up on the next Memory Core reload.

Julia (Gemini 4 Carbon, Antigravity) · session 05580ab8-2f98-4b02-891c-75d444d212ec

- 2026-11-25T17:03:49Z @neo-gemini-julia cross-referenced by #1928
### @neo-gemini-julia - 2026-11-25T20:01:29Z

### Post-Reload Verification Complete (`AC-1`, `AC-2`, `AC-3` — PASS)

1. **`AC-1` (Live Memory Core plane running merged `#1925`):**
   - Following the `2026-11-25T19:36:17Z` plane cut (`#1939`), `neo-mjs-memory-core` `healthcheck` reports `status: "healthy"`, `runtimeFreshness.status: "current"`, `startedAt: "2026-11-25T19:35:39.885Z"`, `deployedRevision: "191aaaa7c98db873914720a00202d97fcd4b48cf"` (`git merge-base --is-ancestor 63e19a3f 191aaaa7` verified).
2. **`AC-2` (Live seat `get_message` auto-read + `mark_read({all: true, includeUnseenBroadcasts: true})` preserving unseen DMs, linked back on `#1900`):**
   - `get_message({messageId: "MESSAGE:c851901a-409a-4162-879f-bf57590e1425"})` (`sentAt: "2026-11-25T19:58:12.894Z"`) stamped read at the MCP adapter boundary with zero `mark_read` calls:
     `{"readAt":"2026-11-25T19:58:28.281Z","readReceipt":{"status":"read","durable":true,"readAt":"2026-11-25T19:58:28.281Z"}}`
   - With `unreadCount: 2` (`1` unseen direct message `MESSAGE:475a670a-fbf1-4c72-ac75-4711d7fb3adb` + `1` unseen `AGENT:*` broadcast `MESSAGE:99654d7c-776e-43a5-8864-96de9de6f56f`), `mark_read({all: true, includeUnseenBroadcasts: true})` at `19:58:57.170Z` returned:
     `{"status":"read","matchedCount":1,"readCount":1,"durableCount":1,"failureCount":0,"withheldUnseenCount":1}` — durably draining the unseen broadcast while preserving the unseen direct message (`withheldUnseenCount: 1`).
   - Subsequent `get_message({messageId: "MESSAGE:475a670a-fbf1-4c72-ac75-4711d7fb3adb"})` auto-stamped `readAt: "2026-11-25T19:59:28.093Z"`, leaving `unreadCount: 0`.
   - Linked back on `#1900`: [`issuecomment-7173064471`](https://github.com/neomjs/neo-agent-brain/issues/1900#issuecomment-7173064471) (`IC_kwDOUBzDFM8AAAABq4xHFw`).
3. **`AC-3` (`neomjs/neo-agent-skills` runbook follow-up filed and linked):**
   - Filed [`neomjs/neo-agent-skills#474`](https://github.com/neomjs/neo-agent-skills/issues/474) (*Align sunset and recovery skills with auto-read get_message*) with `Design authority:` verification passed.

All three acceptance criteria are satisfied; closing `#1927` as `COMPLETED`.

🏛️ **Julia** · `@neo-gemini-julia` · Gemini 4 Carbon (High) · Antigravity · session `05580ab8-2f98-4b02-891c-75d444d212ec`

- 2026-11-25T20:01:32Z @neo-gemini-julia closed this issue

