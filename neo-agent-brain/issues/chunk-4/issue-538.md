---
id: 538
title: 'Rebuild the message edges the GC and the decay removed, from the message record'
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-26T07:36:53Z'
updatedAt: '2026-09-26T07:36:53Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/538'
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
---
# Rebuild the message edges the GC and the decay removed, from the message record

## Context

On the local plane (read-only, 2026-09-26 07:1xZ): 3,565 `MESSAGE` nodes carry `inReplyTo`, and 609 `IN_REPLY_TO` edges exist; 206 carry `partOfThread`, and 5 `PART_OF_THREAD` edges exist; `REFERENCES_TICKET` 6, `TAGGED_CONCEPT` 4,014 across 28,800 messages since 2026-06. Two removers: the #506 GC severed ~140k "unanchored" edges on 09-25 and the mailbox projection re-derives only the three carriers; and the ambient decay ticking ~4×/day (#537) pruned message edges below weight 0.2. Grace's 07:14Z receipt and the operator's escalation are the sightings.

## The Problem

The six optional edges are written only by the full projection (`MailboxService._projectMessageWalRecord` without `onlyIssues`, `ai/services/memory-core/MailboxService.mjs:2870-2889`). `repairMessageGraphIntegrity` scans the three required carriers, so nothing rebuilds a missing optional edge. `list_messages` resolves `threadId`, `inReplyTo`, `relatedTickets` and `taggedConcepts` through those edges (`MailboxService.mjs:3386`), so old threads and ticket filters are empty for most of the mailbox.

## The Architectural Reality

- The `MESSAGE` node carries the declared fields the edges are derived from: `inReplyTo`, `partOfThread`, `relatedTickets`, `taggedConcepts`, `originSessionId`, `relatedSessions`. The record is the source of truth; the WAL is not needed.
- `GraphService.linkNodes` culls an edge whose endpoint is absent (the FK guard) and reinforces an existing edge's weight by 0.1, so a rebuild links only where the target node exists and no `(source, target, type)` row exists.
- After #537, edges a message sources are exempt from decay; before it they decay again. Sequence: after #537 runs on the plane.

## The Fix

A maintenance entry under `ai/scripts/maintenance/` (a mode beside #509's receipts CLI or its own file, decided at the PR; siblings `restore.mjs`, `backup.mjs`): for every `MESSAGE` node and each declared field, link the edge at weight 1.0 when the target exists and the row is missing. Report `{messages, linked, present, missingTarget}`. Dry-run by default; `--apply` writes.

**Contract Ledger**

| Target surface | Authority | Proposed behavior | Fallback | Evidence |
|---|---|---|---|---|
| the maintenance entry (new) | this ticket; `MailboxService`'s optional-edge projection | rebuilds missing optional edges from the node's fields, idempotent | refuses to write without `--apply`; never touches the carriers | unit arms below; live counts on this ticket |
| `IN_REPLY_TO` / `PART_OF_THREAD` / `REFERENCES_TICKET` / `TAGGED_CONCEPT` / `ORIGINATES_IN` / `RELATED_SESSION` (existing) | `MailboxService` | unchanged semantics; missing rows restored at weight 1.0 | — | before/after counts |

**Decision Record impact:** none; the edge model is neo #11029's.

## Acceptance Criteria

- [ ] AC-1: the dry run on the live plane reports the counts on this ticket and writes nothing (edge counts per type unchanged).
- [ ] AC-2: unit arms on a real SQLite graph: a missing `IN_REPLY_TO` and `TAGGED_CONCEPT` are linked at 1.0; a present edge keeps its weight; a missing target is reported, not linked; the dry run writes nothing; a rerun links nothing.
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

