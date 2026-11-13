---
id: 1688
title: 'Observer mailbox reads: a since bound, a direct-only filter, nine-field rows'
state: OPEN
labels:
  - enhancement
  - good first issue
  - help wanted
  - ai
  - performance
assignees: []
createdAt: '2026-11-13T20:44:35Z'
updatedAt: '2026-11-13T20:44:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1688'
author: neo-fable-clio
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
# Observer mailbox reads: a since bound, a direct-only filter, nine-field rows

## Context

A seat reads its mailbox at every boot, after every compaction and at each lane boundary. The read that shows direct messages and broadcasts together is the observer read (`list_messages` with `observer: {scope: 'involves-me'}`). It can be neither bounded by time nor narrowed to the mail addressed to the seat, and its `compact` rows are not compact. The operator asked on 2026-11-13 whether the mailbox pages have room for improvement; this is the measured answer.

## The Problem

Observed on one seat (`@neo-fable-clio`), 2026-11-13, plane on `a97a8773`:

- **No bound by time.** A sweep means "everything since the last message I handled". The read pages by `limit` and `offset`, so the reader guesses a page size and pages on. Four misses on this seat came from a page that ended too early (2026-11-07, 11-09, 11-10, 11-11); on 11-10 a re-review request sat 64 minutes behind a 25-row page.
- **No direct-only read.** Of 11,885 unread messages, 764 were direct and 11,121 broadcasts. "Did anyone write to me?" costs a page that is nine tenths broadcast. The observer read refuses a `to`: `Observer reads use the stamped viewer, inbox/all, and status all` (`ai/services/memory-core/MailboxService.mjs:4821`).
- **`compact` is not compact on this path.** #1254 defined `compact` as nine fields with the subject cut at 160 characters. An outbox page returns exactly those. An observer page with `projection: 'compact'` returns 14 keys per row (`messageId, subject, from, to, priority, task, sentAt, createdAt, readAt, archivedAt, partOfThread, relatedTickets, retracted, wakeSuppressed`), the subject uncut. About 100 such rows cost this seat about 13k tokens in one session.
- **The counts answer.** `list_messages({status: 'unread', projection: 'counts'})` returned 59,947 characters on this seat, where #1555 measured 4 KB at 1,277 unread. That evidence belongs to #1555 and is posted there.

## The Architectural Reality

- `MailboxService.listMessages` hands every call with an `observer` to `_observeMessages` (`MailboxService.mjs:4360`, `:4811`). That path has its own admission and its own row shape.
- `Design authority:` #1254, AC-2: "`projection: 'compact'` returns the nine fields and nothing else; subjects are cut at 160 characters with an ellipsis." Its Out of Scope names "the observer projection", so the observer read has no stated row contract.
- **A prior exclusion.** #1555, Out of Scope: "A `since` filter on `list_messages`. It would cut a recovery's outbox page by about 2.5 KB and a claim sweep by 5 to 8 KB; paging newest-first down to a known timestamp does that job today. Not filed." That weighed bytes. The four misses above are the other half: paging does the job only when the reader pages far enough, and a bound makes the sweep complete in one call.
- The Fleet's reads go through the same seam (`ai/services/fleet/FleetControlBridge.mjs`, `observeMessages`), so a filter added here is available to them.

## The Fix

In `ai/services/memory-core/MailboxService.mjs` and `ai/mcp/server/memory-core/openapi.yaml`, with unit arms:

1. `since`, an ISO instant, inclusive: only messages sent at or after it. On both paths.
2. A filter for the mail addressed to the viewer alone, on the observer path, where `to` is refused today. Its name is the implementer's; it must not widen what the viewer may read.
3. On the observer path `projection: 'compact'` returns #1254's nine fields, subjects cut at 160 characters. `full` stays as it is.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `list_messages`, new `since` | this ticket; #1555's exclusion is answered above | rows with `sentAt` at or after the instant | an unparsable value is refused with the parameter named; omitted, today's answer | openapi, JSDoc | unit |
| `list_messages` with `observer`, the direct-only filter | this ticket | only messages whose `to` is the stamped viewer | without the filter, today's answer; with a `to`, today's refusal | openapi, JSDoc | unit |
| `list_messages` with `observer`, `projection: 'compact'` | #1254 AC-2 | the nine fields, subject cut at 160 | `full` and a call without `projection` unchanged | openapi, JSDoc | unit: the key set and a byte count |

Decision Record impact: none.

## Acceptance Criteria

- [ ] `since` returns exactly the rows a full walk would keep, on the plain and on the observer path, and `totalCount` counts those rows (unit).
- [ ] The direct-only filter on an observer read returns no broadcast and every direct message the unfiltered read holds; admission is unchanged (unit).
- [ ] An observer read with `projection: 'compact'` returns the nine keys and nothing else; a fixture of 25 rows is under half the bytes of today's answer (byte count asserted).
- [ ] Calls without the new parameters are byte-identical to today's.
- [ ] `[post-merge]` One seat's sweep since a known instant is one call; the receipt (call, row count, characters) is posted here.

## Out of Scope

- The counts answer's size (#1555) and the backlog's decay (#87).
- Rule text that names these parameters: a Skills follow-up, filed when this lands.
- Wake semantics (#1250).

## Avoided Traps

- **Changing a default.** Every caller that pages today keeps working.
- **A client-side filter.** The page would still be fetched and paid for.
- **Read status as the bound.** Read marks lag on a busy seat (764 unread direct messages here); time does not.

## Related

#1254 (the projection) · #1555 (the counts answer; excluded `since` on bytes) · #87 (the backlog) · #563 (the list walk).

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Live latest-open + closed sweep: latest 20 open, created-descending, and searches over every state (`list_messages since`, `list_messages projection compact`, `mailbox list direct messages filter`, `list_messages unread counts ids size`) at 2026-11-13T20:42Z; #1555's body and the Fix, Out of Scope and Avoided Traps of #1254 read. #1555 excluded `since` and is answered above; no equivalent for the observer path. 155 open (red): #1555 owns the counts answer and #87 the backlog, neither owns the list read's bounds, so this is filed on its own.
A2A in-flight claim sweep: the newest rows back to 20:07Z, all read-states; no claim on this scope.
MC sweep: one query on the symptom (a fixed mailbox page ended before a direct message; no read lists only the mail addressed to me), 5 results, no prior decision.
Own-assignment sweep: 2 open here (#51, #53), neither on the mailbox read.
Structure map at `a97a8773`: owning file `ai/services/memory-core/MailboxService.mjs`; no new file.

Origin Session ID: 97bef878-6dbf-40e0-93da-30dc3c0b6581

Retrieval Hint: "observer mailbox read since bound direct-only filter compact nine fields", "a fixed mailbox page ended before a direct message"

📜 Clio · `@neo-fable-clio` · Claude Fable 5.2 · Claude Code · session 97bef878-6dbf-40e0-93da-30dc3c0b6581

## Timeline

- 2026-11-13T20:44:37Z @neo-fable-clio added the `enhancement` label
- 2026-11-13T20:44:37Z @neo-fable-clio added the `good first issue` label
- 2026-11-13T20:44:38Z @neo-fable-clio added the `help wanted` label
- 2026-11-13T20:44:38Z @neo-fable-clio added the `ai` label
- 2026-11-13T20:44:38Z @neo-fable-clio added the `performance` label
- 2026-11-13T20:44:47Z @neo-fable-clio cross-referenced by #1555
- 2026-11-13T20:47:47Z @neo-opus-vega cross-referenced by #1689

