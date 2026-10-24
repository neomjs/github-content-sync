---
id: 1254
title: 'list_messages gains a compact projection, so a boot''s mailbox check costs counts and subjects, not pages of rows'
state: OPEN
labels:
  - enhancement
  - ai
  - performance
  - agent-os
assignees: []
createdAt: '2026-10-24T20:15:14Z'
updatedAt: '2026-10-24T20:15:14Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1254'
author: neo-fable-mnemosyne
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
# list_messages gains a compact projection, so a boot's mailbox check costs counts and subjects, not pages of rows

## Context

Every recovery starts with `list_messages({status: 'unread'})`. Measured 2026-10-24 on a Claude seat: 127 unread rows, all but two of them `AGENT:*` broadcasts nobody drains; one page of 50 rows came back as ≈ 36 KB of JSON (≈ 9k tokens), the outbox page of 15 another ≈ 5k. The two rows that mattered were direct messages. The backlog does not shrink: broadcasts are FYI by design and no step marks them read, so the same 9k is paid at every boot by every seat.

## The Problem

The listing has one shape, the full row: ids, subject, priority, timestamps, sender class, recipient, wake flag, related tickets, tagged concepts, task envelope. The recovery runbook needs two things from it: how many unread and of what kind, and the direct messages in full. Everything else on the page is read once and discarded. There is no projection parameter and no counts-only answer, so a seat pays for 50 rows to learn that two were for it.

## The Architectural Reality

- `ai/services/memory-core/MailboxService.mjs:3658` `listMessages(args, {recordSeen})` returns a page `{messages, totalCount, truncated, nextOffset}`; `limit` defaults to 50; a `limit` of 0 is rejected rather than clamped (the continuation promise). The row shape is the stored message minus the body.
- `mark_read({all: true})` drains only rows a previous listing surfaced (`openapi.yaml:2849`); `includeUnseen: true` widens it. So a drain exists, but the runbook never calls it, and a drain of broadcasts a seat never read is exactly what `withheldUnseenCount` guards against.
- The OpenAPI operation `list_messages` is at `openapi.yaml:2584`; `list_issues` on the GitHub server already has `projection: full | summary | title | title_only` — the precedent for the shape.
- Design authority: the method's JSDoc ("a PAGE, never a set") documents full rows as intended; this is a contract addition on the measured cost, not a defect claim.

## The Fix

1. `projection` on `list_messages`: `full` (today, stays the default so no caller changes), `compact` (per row: `messageId, from, to, sentAt, readAt, wakeSuppressed, priority, taskState, subject` truncated to 160 characters; no tags, no related tickets, no task envelope), `counts` (no rows: `totalCount`, `unreadCount`, by `to` class (`direct` vs `broadcast`), by sender, by first subject tag such as `[lane-claim]`, `[pr-opened]`, `[merged]`, plus the `messageId`s of unread **direct** messages so the runbook can `get_message` them without a page).
2. The recovery runbook's mailbox step becomes: `counts` first, then `get_message` on the direct ids, then `compact` on the broadcast page only when the counts show something unusual (a neo-agent-skills follow-up, filed when this lands).
3. The session-sunset step gains a broadcast drain: `mark_read({all: true})` after the seat's own listing — a skills follow-up too; this ticket only makes the projection exist and keeps `withheldUnseenCount` semantics untouched.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `list_messages` (MCP, `openapi.yaml:2584`) | the operation's description + `MailboxService.listMessages` JSDoc | `projection` enum, default `full` | unknown value rejected with the enum named; `counts` ignores `limit`/`offset` and says so | openapi + JSDoc | unit arms: `full` byte-identical to today; `compact` row shape; `counts` totals equal a `full` walk |
| `recordSeen` stamping | the second-argument contract (`:3658` JSDoc) | `compact` stamps `seenAt` like `full` (rows were displayed); `counts` stamps nothing (no row was shown) | — | JSDoc | arm: `counts` leaves `seenAt` null; `mark_read all` after `counts` drains nothing |
| the observer path (`_observeMessages`) | its own admission | unchanged | — | — | existing arms |

Decision Record impact: none.

## Acceptance Criteria

- [ ] AC-1 `projection: 'counts'` on a mailbox of 130 unread returns under 1,500 characters with the per-class counts and the unread direct-message ids (unit arm on a seeded fixture; byte count asserted).
- [ ] AC-2 `projection: 'compact'` returns the nine fields and nothing else; subjects are cut at 160 characters with an ellipsis.
- [ ] AC-3 `projection: 'full'` and a call without the parameter are byte-identical to today's page.
- [ ] AC-4 `counts` does not stamp `seenAt`; a subsequent `mark_read({all: true})` drains nothing new (the withheld-unseen guard holds).
- [ ] AC-5 (post-merge) a fresh seat's recovery runs `counts` + the direct reads under 2k tokens where today's page costs ≈ 9k (receipt on this ticket); the runbook and sunset follow-ups are filed with their numbers.

## Out of Scope

Changing wake semantics (#1250 owns that); auto-expiring broadcasts; the observer projection; the Fleet's activity feed.

## Avoided Traps

- Changing the default: every caller that walks pages today keeps working.
- Counting read-status as relevance: the runbook still reads direct messages regardless of `readAt`; the claim sweep in `ticket-create` still lists `status: 'all'`.
- A drain that marks unseen mail read: `counts` shows nothing, so it stamps nothing.

## Related

Sibling filings tonight: the Golden-Path-only `get_sandman_handoff` default, the compact `get_context_frontier` default, the boot Focus section (sub of #123), the skills description budget (neo-agent-skills); #1250 (wake semantics, the same service); the context-recovery runbook (neo-agent-skills) for the follow-ups.

unowned-rationale: a one-PR leaf any seat can take; the filer's week is committed (the business block, outcomes 3, 7, 8); the measurement in Context is the premise.

Live latest-open sweep: the latest 20 open issues, created-descending, at 2026-10-24 19:5xZ; no equivalent. A2A in-flight claim sweep: the latest 30 `[lane-claim]` rows, none on this scope. MC sweep: problem-noun queries on the boot cost, 6 results, initialization noise, no prior decision found. Own-assignment sweep: 11 open, #1250 is the same service and a different concern. Structure map: `ai/services/memory-core` (MailboxService), `ai/mcp/server/memory-core` (openapi) — owning folders exist; no new file.

Origin Session ID: 542d408d-b0eb-497c-8f57-a19ca5833f5a

Retrieval Hint: "list_messages projection counts compact unread broadcasts boot mailbox check cost"

## Timeline

- 2026-10-24T20:15:16Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-10-24T20:15:16Z @neo-fable-mnemosyne added the `ai` label
- 2026-10-24T20:15:16Z @neo-fable-mnemosyne added the `performance` label
- 2026-10-24T20:15:17Z @neo-fable-mnemosyne added the `agent-os` label

