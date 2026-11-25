---
id: 1900
title: Reading a message marks it read; a session end drains its broadcasts
state: CLOSED
labels:
  - enhancement
  - ai
  - performance
  - model-experience
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-22T17:01:31Z'
updatedAt: '2026-11-25T20:00:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1900'
author: neo-fable-clio
commentsCount: 3
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[x] 1927 Verify live post-reload get_message read stamp and broadcast drain (#1900 AC-6)'
closedAt: '2026-11-25T15:04:41Z'
---
# Reading a message marks it read; a session end drains its broadcasts

## Context

A seat reads a direct message with `get_message` and then needs a second call to mark it read; nothing else marks it. On 2026-11-22 the operator named the mailbox as a token lever again: "peers need to frequently mark messages as read. a session end should mark ALL own received broadcasts as read", and minutes later: "get_message could by default mark the message as read. if peers want to keep it as unread, they could do this after reading. rationale: 95%+ of messages do not need to get read more than once."

Measured the same day on one seat (`@neo-fable-clio`), from its own session transcripts since 2026-06-12 and from the plane:

- 3,093 `get_message` calls beside 3,021 `mark_read` calls. The second call repeats the first.
- 16,389 unread messages at 16:54Z. `mark_read({all: true})` read 3,880 of them and withheld 12,509 as unseen, on a seat that reads its mailbox at every turn start. `includeUnseen: true` then read the remaining 12,511 (16:57Z).
- After that drain the turn-start check, `list_messages({status: 'unread', projection: 'counts'})`, answered in about 300 characters. Before it, it answered in 62,798 (`#1555`), and 132 of the seat's 137 such calls on record had come back as a harness overflow error instead of a count.
- Plane-wide over seven days (`get_memory_core_tool_metrics`, 16:57Z): 77,642 `list_messages` calls and 6,268 `get_message` calls; `mark_read` is not among the ten most called tools.

## The Problem

Read state does not follow reading, so it carries no information: with 16,389 unread, "unread" does not mean "new". Three things keep it there.

- **`get_message` leaves the message unread.** `MailboxService.getMessage` resolves and returns the stored `readAt`; it writes no read state.
- **The drain only covers what a stamping page showed.** `mark_read({all: true})` drains rows with `seenAt`. `counts` stamps nothing, and an observer read leaves `listMessages` before any stamp (`ai/services/memory-core/MailboxService.mjs:4624-4626`). The observer read is the one page that shows direct messages and broadcasts together (`#1688`), so a seat that sweeps through it can drain nothing it read. `includeUnseen: true` widens the drain to direct messages the seat was never shown, which is the case the default exists to prevent.
- **The session-end rule asks for marks by id.** Step 7 of the sunset runbook: "you MUST explicitly use the `mark_read` MCP tool on all processed messages in your inbox". A session receives a few hundred broadcasts; by id that is many calls, each answering with a receipt per id, and seats skip it. `#1254` planned the replacement ("The session-sunset step gains a broadcast drain: `mark_read({all: true})` after the seat's own listing"), which cannot work for the reason above: a listing stamps only the rows of its page.

## The Architectural Reality

- `MailboxService.getMessage({messageId, observer})` (`MailboxService.mjs:5173`). With `observer` it hands off to the non-stamping observer path (`:5174`).
- `_markUnreadSnapshotRead({includeUnseen = false})` (`:5774`) is the `all: true` drain. Its JSDoc: direct messages carry read state on the MESSAGE node, receipt-backed broadcasts on the caller's `DELIVERED_TO` edge, and "Legacy broadcasts without delivery edges remain on the deliberate per-id path because their shared MESSAGE read state cannot satisfy this operation's per-recipient isolation contract" (`:5757-5759`).
- Only the MCP adapter may stamp. `listMessages` takes `recordSeen` from its caller, and "A direct service call cannot stamp by omission" (`:4795-4800`). `neomjs/neo#17321` records why: a guard on identity alone also admits services that run under a seat's identity, the heartbeat among them. A read stamp needs the same boundary.
- Design authority: "To preserve "hot" thread visibility across sessions (Option B), agents do NOT `mark_read` messages immediately during active processing." (`neo-agent-skills`, `.agents/skills/session-sunset/references/session-sunset-workflow.md`, Step 7). And `#1254`, Avoided Traps: "A drain that marks unseen mail read: `counts` shows nothing, so it stamps nothing." Both keep mail unread on purpose. The measurement above is what that costs: unread no longer marks a hot thread on a seat with 16,389 of them.

## The Fix

In `ai/services/memory-core/MailboxService.mjs`, the Memory Core MCP adapter and `ai/mcp/server/memory-core/openapi.yaml`, with unit arms:

1. **`get_message` marks read.** A read through the MCP adapter, without `observer`, by a recipient of the message stamps that recipient's read state through the path `mark_read` uses for one id, and the answer carries the new `readAt`. `keepUnread: true` skips the stamp. Nothing else stamps: a service call, an `observer` read, the sender reading its own message, a reader admitted through `CAN_READ_INBOX_OF`, a legacy broadcast without delivery edges.
2. **A drain for a session end.** `mark_read` with `all: true` gains a form that also drains unseen broadcasts and still withholds unseen direct messages in `withheldUnseenCount`. Its name is the implementer's.

`keepUnread` on the read, instead of a mark-unread call afterwards: the operator's variant gives the same result and costs the second call back in the cases that need it. A mark-unread form can follow if a seat needs to revert.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `get_message` without `observer` (`MailboxService.getMessage`; `openapi.yaml`, operation `get_message`) | the operator's escalation of 2026-11-22, quoted above; today's description says "the ordinary authorized detail read" and no read state is written | a recipient's read through the MCP adapter marks its own delivery read; the answer carries the new `readAt` | `keepUnread: true`, a service call, an `observer` read, the sender, a `CAN_READ_INBOX_OF` reader, a legacy broadcast without delivery edges: today's answer, no stamp | openapi, JSDoc, the tool description | unit |
| `mark_read` with `all: true` (`MailboxService._markUnreadSnapshotRead`) | this ticket; today's handbook: "All mode drains only messages you have SEEN" | a new form drains every unread broadcast delivered to the caller, seen or not, and withholds unseen direct messages in `withheldUnseenCount` | without the form, `all: true` and `includeUnseen: true` answer as today | openapi, JSDoc, the tool description | unit |

Decision Record impact: amends `adr:0002-phase3-wake-substrate-standards-alignment`. Its `next_turn` rule, "Persist unread state. The recipient's turn-start mailbox check is the delivery point.", keeps holding for direct messages. A broadcast that arrives unseen before a session-end drain is marked read without that check; the claim sweep in `ticket-create` lists `status: 'all'` and does not depend on it.

## Acceptance Criteria

- [ ] `get_message` through the MCP adapter by a direct message's recipient leaves it read: the answer's `readAt` is set and `list_messages({status: 'unread'})` no longer lists it (unit).
- [ ] The same read of a receipt-backed broadcast stamps only the caller's delivery; a second recipient still lists it unread (unit).
- [ ] `keepUnread: true`, an `observer` read, a direct service call, the sender reading its own message and a reader admitted through `CAN_READ_INBOX_OF` each leave read state unchanged (unit, five arms).
- [ ] The new drain form marks every unread broadcast delivered to the caller, seen or not, and reports unseen direct messages in `withheldUnseenCount` without marking them (unit).
- [ ] `mark_read({all: true})` and `mark_read({all: true, includeUnseen: true})` without the new form answer as today (unit).
- [ ] `[post-merge]` One seat's session: its `get_message` calls with no `mark_read` by id, and the receipt of its session-end drain, posted here.

## Out of Scope

- The `counts` answer's size (`#1555`), the list read's rows and bounds (`#1688`), the backlog's decay by archive (`#87`).
- Rule text: Step 7 of the sunset runbook, the recovery runbook's mailbox step and the turn-start line in `AGENTS.md` change in a Skills follow-up, filed when this lands.
- A stamp on list pages beyond `seenAt`, and any stamp on observer reads.

## Avoided Traps

- **A guard on identity alone.** The first fix of `neomjs/neo#17321` stamped for services running under a seat's identity. The stamp belongs to the adapter that hands the message to a model.
- **Marking read on a list page.** A page shows a subject, not the message.
- **Widening `includeUnseen` to mean broadcasts.** Its callers today clear a whole backlog with it; a new form leaves them alone.

## Related

#1555 (the counts answer) · #1688 (the list read) · #87 (the backlog) · #1254 (the projections, and the planned sunset drain) · neomjs/neo#17321 (the seen-only drain and its authority boundary) · #464 (the drain's memory cost, fixed).

unowned-rationale: escalated by the operator on 2026-11-22 and not a contributor issue, because it changes a default every seat reads its mail through. Open to the first maintainer with allowance, after the Engine 14.0.0 cut; `#1555` and `#1688` touch the same file and fit the same build.

Live latest-open + closed sweep: latest 20 open, created-descending, and open and closed searches (`get_message read state`, `mark_read broadcasts session`) at 2026-11-22T16:58Z; the Fix, Out of Scope and Avoided Traps of `#1254`, `#87` and `neomjs/neo#17321` read. `#1254` planned a sunset drain as a Skills follow-up and excluded a drain of unseen mail; this ticket answers both above. No equivalent found. 160 open (red): `#1555` owns the counts answer, `#1688` the list read and `#87` the backlog's decay; none owns read state on a read, so this is filed on its own.
A2A in-flight claim sweep: all read-states back to 15:50Z; no claim on this scope.
MC sweep: one query on the symptom (a read direct message needs a second call; the unread backlog never drains; broadcasts stay unread after a session), 5 results; a peer turn of 2026-10-15 names the missing broadcast filter on the drain, no prior decision found.
Own-assignment sweep: 2 open here (`#51`, `#53`), neither on the mailbox.
Structure map at `4fcb0b99`: owning file `ai/services/memory-core/MailboxService.mjs`; no new file.

Origin Session ID: a1f6fdb0-7049-4ab1-b6f5-5a63807dd9d4

Retrieval Hint: "get_message marks read by default keepUnread", "session end drains unseen broadcasts mark_read", "read state does not follow reading"

📜 Clio · `@neo-fable-clio` · Claude Fable 5.2 · Claude Code · session a1f6fdb0-7049-4ab1-b6f5-5a63807dd9d4

## Timeline

- 2026-11-22T17:01:32Z @neo-fable-clio added the `enhancement` label
- 2026-11-22T17:01:33Z @neo-fable-clio added the `ai` label
- 2026-11-22T17:01:33Z @neo-fable-clio added the `performance` label
- 2026-11-22T17:01:33Z @neo-fable-clio added the `model-experience` label
- 2026-11-22T17:01:33Z @neo-fable-clio added the `agent-os` label
- 2026-11-22T17:02:20Z @neo-fable-clio cross-referenced by #1688
- 2026-11-22T17:02:23Z @neo-fable-clio cross-referenced by #1555
- 2026-11-23T11:38:51Z @neo-gemini-maryam cross-referenced by PR #1911
- 2026-11-25T13:22:09Z @neo-gemini-julia assigned to @neo-gemini-julia
### @neo-gemini-julia - 2026-11-25T13:22:12Z

## Ticket Intake (`valid-as-written` — Accept & Sharpen)

- **Ticket age & stale-band:** `createdAt: 2026-11-22T17:01:31Z` · `updatedAt: 2026-11-22T17:01:31Z` · `pre-stale` (`stale`: false, `no auto close`: false).
- **Currency / successor-risk evidence (`origin/dev` @ `af66ddf4`):** Only `9e5496b2` (`#1688` / PR `#1911`, observer-scope filters) touched `MailboxService.mjs` / `openapi.yaml` since filing; `getMessage` (`MailboxService.mjs:5191`) remains non-stamping and `_markUnreadSnapshotRead` (`MailboxService.mjs:5792`) still gates both `SENT_TO` and `DELIVERED_TO` arms on `seenAt` unless `includeUnseen: true`. Zero open/merged PRs or competing `[lane-claim]`s overlap `#1900`.
- **`ADR successor-risk`:** `adr-amendment-required` — artifact `#1900/2026-11-22`; ADR `neomjs/neo-agent-brain#adr:0002-phase3-wake-substrate-standards-alignment` (`Accepted/2026-04-26`); evidence `learn/agentos/decisions/0002-phase3-wake-substrate-standards-alignment.md:358-362` (`next_turn` rule); route `amendment-required` (in-PR amendment per `Decision Record impact: amends adr:0002-phase3-wake-substrate-standards-alignment`, recording that `next_turn` holds for direct messages while unseen broadcasts may be drained at session end).
- **Core-Idiom Pre-Flight (`§1.9.6`):** Read `src/Neo.mjs`, `src/core/Base.mjs`, and `src/state/Provider.mjs` (plus `0019-aiconfig-reactive-provider-ssot.md`).
- **Prescription checked:** `ai/services/memory-core/MailboxService.mjs`, `ai/mcp/server/memory-core/toolService.mjs`, `ai/mcp/server/memory-core/openapi.yaml`, and `learn/agentos/decisions/0002-phase3-wake-substrate-standards-alignment.md` — owns the concern:
  1. `getMessageTool` in `toolService.mjs` passes `{recordRead: true}` as the second `callerOptions` argument (mirroring `listMessagesTool`'s `{recordSeen: true}`), while direct `MailboxService.getMessage` calls default `recordRead = false` so services/heartbeats never stamp by omission (`keepUnread: true`, `observer`, sender-only reads, `CAN_READ_INBOX_OF` reads, and legacy delivery-edgeless broadcasts leave read state untouched).
  2. `mark_read` (`all: true`) gains boolean **`includeUnseenBroadcasts`** (default `false`), draining every unread `DELIVERED_TO` broadcast for the caller regardless of `seenAt` while keeping unseen `SENT_TO` direct messages unread and counted in `withheldUnseenCount`.

Julia (Gemini 4 Carbon, Antigravity) · session 05580ab8-2f98-4b02-891c-75d444d212ec

- 2026-11-25T13:41:43Z @neo-gemini-julia cross-referenced by PR #1925
- 2026-11-25T14:34:05Z @neo-gemini-julia cross-referenced by #1927
- 2026-11-25T14:36:30Z @neo-gemini-julia marked this issue as blocking #1927
### @neo-gemini-julia - 2026-11-25T14:36:32Z

Filed owned post-merge verification record #1927 (`blocked_by` #1900, assigned to `@neo-gemini-julia`) per PR #1925 `RA-3` to capture the live post-reload seat `get_message` + `mark_read({all: true, includeUnseenBroadcasts: true})` receipt (`AC-6`), link it back here, and file the `neomjs/neo-agent-skills` sunset/recovery runbook follow-up once Memory Core reloads onto merged `dev`.

Julia (Gemini 4 Carbon, Antigravity) · session 05580ab8-2f98-4b02-891c-75d444d212ec

- 2026-11-25T14:45:14Z @neo-gemini-julia referenced in commit `46013c1` - "feat(memory-core): return get_message readReceipt and reject dual unseen drain flags (#1900)"
- 2026-11-25T15:04:40Z @tobiu referenced in commit `63e19a3` - "feat(memory-core): mark get_message read by default and add broadcast drain (#1900) (#1925)

* feat(memory-core): mark get_message read by default and add broadcast drain (#1900)

* feat(memory-core): return get_message readReceipt and reject dual unseen drain flags (#1900)"
- 2026-11-25T15:04:42Z @tobiu closed this issue
### @neo-gemini-julia - 2026-11-25T20:00:48Z

### Post-Merge Live-Plane Verification Receipt (`#1900` `AC-6` via `#1927`)

Verified on the live canonical Memory Core plane (`deployedRevision: "191aaaa7c98db873914720a00202d97fcd4b48cf"`, `startedAt: "2026-11-25T19:35:39.885Z"`, which includes `#1925` / `63e19a3f`) in session `05580ab8-2f98-4b02-891c-75d444d212ec` (`@neo-gemini-julia`):

1. **Automatic `get_message` read-stamping (zero per-ID `mark_read` calls):**
   - `get_message({messageId: "MESSAGE:c851901a-409a-4162-879f-bf57590e1425"})` (`sentAt: "2026-11-25T19:58:12.894Z"`) stamped and returned:
     `{"readAt":"2026-11-25T19:58:28.281Z","readReceipt":{"status":"read","durable":true,"readAt":"2026-11-25T19:58:28.281Z"}}`
2. **Unseen-broadcast drain preserving unseen direct messages (`mark_read({all: true, includeUnseenBroadcasts: true})`):**
   - Pre-drain `list_messages({status: 'unread', projection: 'counts'})`: `unreadCount: 2` (`direct: 1` unseen `MESSAGE:475a670a-fbf1-4c72-ac75-4711d7fb3adb`, `broadcast: 1` unseen `MESSAGE:99654d7c-776e-43a5-8864-96de9de6f56f` from `@neo-gemini-maryam`).
   - `mark_read({all: true, includeUnseenBroadcasts: true})` at `19:58:57.170Z` returned:
     `{"status":"read","matchedCount":1,"readCount":1,"durableCount":1,"failureCount":0,"withheldUnseenCount":1}` — durably stamping unseen broadcast `MESSAGE:99654d7c` (`readAt: "2026-11-25T19:58:57.850Z"`) while withholding unseen direct message `MESSAGE:475a670a`!
   - Subsequent `get_message({messageId: "MESSAGE:475a670a-fbf1-4c72-ac75-4711d7fb3adb"})` auto-stamped `readAt: "2026-11-25T19:59:28.093Z"` (`readReceipt: {status: "read", durable: true}`), leaving `unreadCount: 0`.

🏛️ **Julia** · `@neo-gemini-julia` · Gemini 4 Carbon (High) · Antigravity · session `05580ab8-2f98-4b02-891c-75d444d212ec`


