---
id: 538
title: 'Rebuild the message edges the GC and the decay removed, from the message record'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-26T07:36:53Z'
updatedAt: '2026-09-26T10:37:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/538'
author: neo-opus-vega
commentsCount: 1
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
closedAt: '2026-09-26T10:16:40Z'
---
# Rebuild the message edges the GC and the decay removed, from the message record

## Context

On the local plane (read-only, 2026-09-26 07:1xZ): 3,565 `MESSAGE` nodes carry `inReplyTo`, and 609 `IN_REPLY_TO` edges exist; 206 carry `partOfThread`, and 5 `PART_OF_THREAD` edges exist; `REFERENCES_TICKET` 6, `TAGGED_CONCEPT` 4,014 across 28,800 messages since 2026-06. Two removers: the #506 GC severed ~140k "unanchored" edges on 09-25 and the mailbox projection re-derives only the three carriers; and the ambient decay ticking ~4×/day (#537) pruned message edges below weight 0.2. Grace's 07:14Z receipt and the operator's escalation are the sightings.

## The Problem

The six optional edges are written only by the full projection (`MailboxService._projectMessageWalRecord` without `onlyIssues`, `ai/services/memory-core/MailboxService.mjs:2870-2889`). `repairMessageGraphIntegrity` scans the three required carriers, so nothing rebuilds a missing optional edge. `list_messages` resolves `threadId`, `inReplyTo`, `relatedTickets` and `taggedConcepts` through those edges (`MailboxService.mjs:3386`), so old threads and ticket filters are empty for most of the mailbox.

## The Architectural Reality

- The `MESSAGE` node carries four of the declared fields the edges are derived from: `inReplyTo`, `partOfThread`, `relatedTickets`, `taggedConcepts` (plus `sentAt` and `userId`, the edge's `timestamp` and `userId`). The two session fields (`originSessionId`, `relatedSessions`) live only on the WAL record (`ai/services/memory-core/MailboxService.mjs:433`), and the message WAL starts at the Brain split (2026-08-26); their absent edges restore from a bundle through #509's CLI (`--edge-types ORIGINATES_IN,RELATED_SESSION`, 206 absent live in the 09-26 dry run), not from the node.
- `GraphService.linkNodes` culls an edge whose endpoint is absent (the FK guard) and reinforces an existing edge's weight by 0.1, so a rebuild links only where the target node exists and no `(source, target, type)` row exists.
- After #537, edges a message sources are exempt from decay; before it they decay again. Sequence: after #537 runs on the plane.

## The Fix

A maintenance entry under `ai/scripts/maintenance/` (its own file beside #509's receipts CLI, which reads a bundle; this one reads the live nodes): for every `MESSAGE` node and each of the four node-borne fields, link the edge the way the projection does — weight 1.0, the message's `sentAt` as `timestamp`, its `userId`, `sharedEntity` — when the row is missing; a tag's concept node is created when absent (`ensureTaggedConceptNode`'s shape), any other target must exist. Report per type `{fields, linked, present, missingTarget, conceptsCreated}`. Dry-run by default (read-only open); `--apply` writes with the identity checked inside the insert.

**Contract Ledger**

| Target surface | Authority | Proposed behavior | Fallback | Evidence |
|---|---|---|---|---|
| the maintenance entry (new) | this ticket; `MailboxService`'s optional-edge projection | rebuilds missing optional edges from the node's fields, idempotent | refuses to write without `--apply`; never touches the carriers | unit arms below; live counts on this ticket |
| `IN_REPLY_TO` / `PART_OF_THREAD` / `REFERENCES_TICKET` / `TAGGED_CONCEPT` (existing) | `MailboxService` | unchanged semantics; missing rows restored at weight 1.0 with the projection's properties | — | before/after counts |
| `ORIGINATES_IN` / `RELATED_SESSION` (existing) | `MailboxService` (WAL-borne) | out of this entry; restored from a bundle by #509's CLI | — | that CLI's dry run |

**Decision Record impact:** none; the edge model is neo #11029's.

## Acceptance Criteria

- [ ] AC-1: the dry run on the live plane reports the counts on this ticket and writes nothing (edge counts per type unchanged).
- [ ] AC-2: unit arms on a real SQLite graph: a missing `IN_REPLY_TO`, `PART_OF_THREAD`, `REFERENCES_TICKET` and `TAGGED_CONCEPT` are linked with the projection's properties; a present edge keeps its weight; a missing target is reported, not linked, and a missing tag concept node is created instead; the dry run writes nothing; a rerun links nothing.
- [ ] AC-3 (post-merge, local plane, after #537 is deployed): after `--apply`, every message whose `inReplyTo` names an existing message has its `IN_REPLY_TO` edge, likewise `partOfThread`; `list_messages({threadId})` on a 2026-07 thread returns its members. Receipt on this ticket.

## Out of Scope

- Receipts (`readAt` / `archivedAt`): #509.
- The three carriers: the projection owns them (#507's guard, #509's restore).
- The decay exemption itself: #537.

## Related

#506 / #507 · #509 · #537 · #517 · neo #11029

Live latest-open sweep: the latest 20 open issues at 2026-09-26T07:3xZ; no equivalent (#509 is receipts, #537 the guard). A2A in-flight sweep: 30 newest at 07:3xZ; no claim. MC sweep: "message edges exempt from ambient decay"; no prior rebuild tool. Own-assignment: #509, #537, #493, #471. Structure map: `ai/scripts/maintenance/` (siblings `restore.mjs`, `backup.mjs`, `compactGraphLog.mjs`).

Origin Session ID: e2fd8a01-5bfc-4d82-ad5b-5c889f4b6710
Retrieval Hint: "rebuild IN_REPLY_TO PART_OF_THREAD TAGGED_CONCEPT edges from MESSAGE node fields after GC and decay removed them"


## Timeline

- 2026-09-26T07:36:54Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-26T07:36:55Z @neo-opus-vega added the `bug` label
- 2026-09-26T07:36:55Z @neo-opus-vega added the `ai` label
- 2026-09-26T07:36:56Z @neo-opus-vega added the `agent-os` label
- 2026-09-26T09:34:45Z @neo-opus-vega cross-referenced by PR #543
- 2026-09-26T09:51:48Z @neo-opus-vega referenced in commit `b5356b9` - "feat(maintenance): the rebuild reads message ids up front so a batch never flushes over an open cursor (#538)"
- 2026-09-26T10:16:40Z @tobiu referenced in commit `6076508` - "Merge pull request #543 from neomjs/vega/538-rebuild-message-edges

feat(maintenance): a message's reply, thread, ticket and tag edges are rebuilt from the message itself (#538)"
- 2026-09-26T10:16:41Z @tobiu closed this issue
- 2026-09-26T10:21:10Z @neo-opus-vega cross-referenced by #64
### @neo-opus-vega - 2026-09-26T10:37:19Z

## AC-3 receipt — `--apply` on the local plane, 2026-09-26 10:35Z (operator's go: both halves)

Brain dev `60f911e`, inside the orchestrator container, navigation half first, then the tag half; 6 s together.

| Type | before | linked | after | missing target (reported, not linked) |
|---|---|---|---|---|
| `IN_REPLY_TO` | 636 | 2,614 | 3,250 | 354 |
| `PART_OF_THREAD` | 5 | 130 | 135 | 71 |
| `REFERENCES_TICKET` | 6 | 313 | 319 | 91,106 |
| `TAGGED_CONCEPT` | 4,144 | 33,851 | 37,995 | 0 |
| `CONCEPT` nodes | 32,274 | 1,898 created | 34,172 | — |

A rerun links 0 replies and 0 tags (1 thread link and 227 ticket links newly linkable: one message arrived meanwhile, and 227 `relatedTickets` strings now match a CONCEPT node the tag half created under the same bare id — a ticket number used as a tag; left unlinked, a ticket reference must not point at a tag; the projection defect-note covers the qualification gap).

`list_messages({threadId: 'MESSAGE:1eae988d-1c7e-4220-9610-158e21116127', box: 'all'})` — an 08-13 thread of six, the oldest this seat is a party to — returns all six members (at most one of its `PART_OF_THREAD` edges existed before: the plane held five in total). AC-3 met.

— Vega (Claude Fable 5.1, Claude Code) 🌿



