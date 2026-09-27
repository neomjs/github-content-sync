---
id: 578
title: 'A landed receipt write reports `not_applied` on a fresh connection: `narrowWriteResult` reads `lastInsertRowid` after the trigger has restored it'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-27T13:15:19Z'
updatedAt: '2026-09-27T14:35:47Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/578'
author: neo-opus-vega
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
closedAt: '2026-09-27T14:35:47Z'
---
# A landed receipt write reports `not_applied` on a fresh connection: `narrowWriteResult` reads `lastInsertRowid` after the trigger has restored it

## Context

Observed on the local plane after the 13:05:59Z recreate onto Brain dev `230593f`: every `mark_read` from 13:07:11Z to 13:09:50Z answered `{status: 'not_applied', durable: false, retryable: true, warning: 'mark_read did NOT apply: the record is no longer present in storage, and the in-memory copy was left unchanged …'}` while the recipient's `DELIVERED_TO` edge in `memory-core-graph.sqlite` carried exactly that call's `readAt` and the caller's unread count fell with each call. At 13:12:04Z — after the in-process drain had projected two new messages, i.e. after the connection's first plain `INSERT` — the same call answered `{status: 'read'}`. Reproduced offline with `better-sqlite3` on a file-backed database with an `AFTER UPDATE` trigger into `GraphLog`: a fresh connection's first narrow `UPDATE` returns `changes: 1, lastInsertRowid: 0`; after one plain `INSERT` it returns `lastInsertRowid: 3` while the trigger's own `GraphLog` row is `4` — a wrong id that happens to be truthy. Defect-note fingerprint `f4045afbb9af3d6c` (13:13Z).

## The Problem

`ai/graph/storage/SQLite.mjs` `narrowWriteResult(result)` returns `result.changes > 0 ? Number(result.lastInsertRowid) : 0` and documents that the `node_update` / `edge_update` triggers "insert a GraphLog row inside the statement, so `lastInsertRowid` names exactly this write's log position". SQLite restores `last_insert_rowid()` to its pre-trigger value when the trigger program ends (`sqlite3_last_insert_rowid()` documentation), so after the `UPDATE` the value is the connection's last PLAIN insert: `0` on a connection that has not inserted yet, a stale rowid otherwise. #20's falsified-premise section records this mechanism (2026-08-23, found by @neo-gpt at intake). What it does not record is the receipt-facing symptom: `MailboxService.writeReceiptField` uses the value as the landed flag (`if (!logId) return writeOnce ? alreadySet : missingRow`), and `receiptWithDurability` turns `missing-row` into `not_applied … the record is no longer present in storage` — a message telling a seat to retry a write that landed. Every recreated mc-server serves that answer for `mark_read`, `archive_message` and the SEEN stamps until its first projected message.

## The Architectural Reality

- `ai/graph/storage/SQLite.mjs:10–24` `narrowWriteResult`; `:499–533` `setRecordPropertyIfAbsent` / `setRecordProperty` — their JSDoc already says `@returns {Boolean}`, the contract the id replaced; `ai/graph/storage/Base.mjs:60/81` the abstract signatures.
- `ai/services/memory-core/MailboxService.mjs:2455–2500` `writeReceiptField` — the only consumer; it reads the value as truthiness and then calls `db.acknowledgeLocalMutations()` (the global-max ack #20 retires). No caller uses the id for a narrow acknowledgement: that strategy (`logId === lastSyncId + 1`) was retracted on #20 as run-order dependent and superseded by writer identity on the row.
- The existing `> 0` unit is vacuous under either hypothesis (#20's Contract Ledger names it); the fresh-connection case is the one assertion the current implementation fails.
- Structure map: owning folder `ai/graph/storage`; spec siblings `test/playwright/unit/ai/graph/SQLiteWriteGuard.spec.mjs`, `Database.spec.mjs` (the storage specs sit beside the graph specs); no new source file.

## The Fix

1. `narrowWriteResult` reports what a narrow write can know: `result.changes > 0` — a Boolean, restoring the documented contract of `setRecordProperty` / `setRecordPropertyIfAbsent`. Its JSDoc names the trigger-exit restoration as the reason the id was never reliable and points to #20 for the writer-identity mechanism.
2. `writeReceiptField` keys `written` / `already-set` / `missing-row` on that Boolean; the `RECEIPT_WRITE` vocabulary and the receipt shapes are unchanged.
3. Arms: (a) storage — a file-backed database reopened on a FRESH connection: the first `setRecordProperty` on an existing row returns `true` and the row carries the value (red today: `0`); `setRecordPropertyIfAbsent` returns `true` once and `false` on the second call; an unknown id returns `false`. (b) mailbox — `markRead` reports `status: 'read'` with no `warning` for a landed write and `not_applied` only when the row is gone (the existing receipt arms stay green).
4. Receipt (L4): the first `mark_read` on the next recreated mc-server answers `read`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `setRecordProperty` / `setRecordPropertyIfAbsent` return | `SQLite.mjs` | Boolean: a row matched and was updated | `false` when the id is absent | JSDoc (already Boolean) | the fresh-connection arm |
| `mark_read` / `archive_message` / SEEN receipts | `writeReceiptField` → `receiptWithDurability` | `read` / `archived` when the row updated; `not_applied` only when no row matched | vocabulary unchanged | tool descriptions unchanged | the mailbox arm + the plane receipt |
| own-row acknowledgement | #20 | untouched: the global-max ack stays until #20's writer identity lands | — | a comment pointing at #20 | — |

## Acceptance Criteria

- [ ] AC-1: on a fresh connection to a file-backed database, the first narrow write on an existing row returns `true` and the value is in the row (red first against today's `0`); the write-once variant returns `true` once and `false` after; an unknown id returns `false`.
- [ ] AC-2: `markRead` / `archiveMessage` on the real service report `read` / `archived` for a landed write and `not_applied` only for a missing row; the existing receipt arms stay green.
- [ ] AC-3 (post-merge, L4; receipt on this ticket): the first `mark_read` on a recreated mc-server answers `status: 'read'`.

## Out of Scope

- The own-row acknowledgement and the retirement of the global-max ack (#20, its writer-identity mechanism).
- Consumers of `getLatestLogId`.

## Related

#20 (the parent mechanism, @neo-opus-grace), neomjs/neo#17511 (where the narrow receipts were introduced), #564 (the cut that exposed it), #64 (today's cut receipts).

Live latest-open sweep: latest 20 open Brain issues at 2026-09-27T13:12Z hold no equivalent (#20 is the acknowledgement mechanism; this is its receipt symptom with a fix that does not wait for it). Exact sweeps: `lastInsertRowid` / `narrowWriteResult` → #20 only; `missing-row` → unrelated hits. A2A in-flight sweep (last 30, any read state): no claim on receipts or the storage primitive. MC sweep: @neo-opus-grace's 2026-08-23 falsification memory (the mechanism recorded; the receipt symptom not). Own-assignment sweep: 7 open, none on `ai/graph/storage`. Structure map: `ai/graph/storage`; no new file.

Decision Record impact: none (#20's ledger stands; this narrows a return value to what it can honestly say).

Origin Session ID: 574ae0b8-b8d0-40d3-8cf6-1693ec48674a
Retrieval Hint: "mark_read not_applied missing-row while readAt landed; narrowWriteResult lastInsertRowid 0 after trigger on a fresh connection; SQLite restores last_insert_rowid at trigger exit"


## Timeline

- 2026-09-27T13:15:19Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-27T13:15:21Z @neo-opus-vega added the `bug` label
- 2026-09-27T13:15:21Z @neo-opus-vega added the `ai` label
- 2026-09-27T13:16:27Z @neo-opus-vega added the `agent-os` label
- 2026-09-27T13:23:21Z @neo-opus-vega cross-referenced by PR #580
- 2026-09-27T13:24:00Z @neo-opus-vega cross-referenced by #64
- 2026-09-27T14:35:47Z @tobiu referenced in commit `b7de5a7` - "Merge pull request #580 from neomjs/vega/578-narrow-write-boolean

fix(graph): a narrow write reports the row it updated, not lastInsertRowid — a landed receipt no longer reads as a missing row on a fresh connection (#578)"
- 2026-09-27T14:35:47Z @tobiu closed this issue

