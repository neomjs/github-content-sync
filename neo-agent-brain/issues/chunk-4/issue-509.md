---
id: 509
title: 'Restore the receipts the GC cycles deleted, from the 13:12Z bundle'
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-25T20:45:31Z'
updatedAt: '2026-09-25T20:45:39Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/509'
author: neo-opus-vega
commentsCount: 0
parentIssue: 64
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
# Restore the receipts the GC cycles deleted, from the 13:12Z bundle

## Context

On the local plane, four REM cycles today (16:09Z, 18:00Z, 18:09Z, 19:51Z) ran the garbage collection #506 names: `Severed 58997 / 29298 / 13370 / 38316 unanchored edges` (the orchestrator's plane log). #507 (merged 20:35Z, dev `7b04b98`, the plane recreated on it 20:41Z) stops the deleter. Nothing restores what it deleted.

## The Problem

The severed edges were real storage deletions (they were cached, so `removeEdges` fired). The mailbox projection re-derived the `DELIVERED_TO` edges from the message WAL, but a re-derived edge starts `readAt: null` with a new id, so every `mark_read` committed before a cycle is gone: live at 20:31Z, 1,813 `DELIVERED_TO` rows carry a `readAt` and 63,326 do not; a seat's unread count reads in the thousands (#506's symptom). Small edge types that no projection re-derives are simply absent (`AGENT_TURN_PRESENCE`: 1,932 in the bundle, below the live top-14's 584 floor).

Nodes and vectors were not lost: `Database.removeNode` reaches storage only through `onNodesMutate(removedItems)`, so the ~129k "orphans" the cycles reported were never deleted (231,812 nodes live), and the vector purge targets collections that hold only protected labels (#506, my 20:25Z comment).

## The Architectural Reality

- **Sources.** `~/.neo-ai/backups/backup-2026-09-25T13-12-54.234Z` (8.1 GB, `restorable: true`, `graph/graph-backup-….jsonl` 391,187 records) predates every cycle; its `DELIVERED_TO` records carry 2,276 non-null `readAt` values. `~/.neo-ai/diagnostics/incident-506/2026-09-25/memory-core-graph.sqlite-wal.snapshot` (2.75 GB, copied 20:09Z) holds every frame from the 18:16Z WAL restart to 20:09Z, so marks committed 18:16–19:51Z and deleted at 19:51Z are in it; marks committed 13:12–18:16Z and deleted at 18:00Z / 18:09Z are in neither (that WAL generation was checkpointed and reset at 18:16Z).
- **The existing restore is the wrong shape for this.** `ai/scripts/maintenance/restore.mjs` `--mode merge` inserts graph rows with `INSERT OR IGNORE` keyed by id and refuses logical duplicates before writing; a re-derived edge has a new id for the same `(source, target, type)`, so a merge either refuses or would duplicate the delivery edge. `--mode replace --preserve-read-state` truncates and rebuilds the graph from the bundle, keeping only the current non-null receipts, which discards seven hours of nodes and edges written since 13:12Z.
- **Receipt custody.** Broadcast receipts live on the per-recipient `DELIVERED_TO` edge (`readAt`, `archivedAt`); direct-message receipts on the `MESSAGE` node (`MailboxService` `writeReceiptField`). Only null-in-live values may be filled, so a fresher live mark is never regressed (the rule `--preserve-read-state` already states for the replace path).

## The Fix

1. A narrow maintenance CLI beside `restore.mjs` (`ai/scripts/maintenance/restoreReceipts.mjs`, dry-run first): for every `DELIVERED_TO` edge in a bundle's graph JSONL with a non-null `readAt` or `archivedAt`, find the live edge by `(source, target, type)` and set the field where live is null; for `MESSAGE` nodes, the same for the node's receipt fields. Report `{matched, filled, alreadySet, missingLive}` and refuse to write outside `--apply`.
2. The same CLI restores edges of types no projection re-derives (`AGENT_TURN_PRESENCE` and the other small types absent live) when no live edge with the same `(source, target, type)` exists and both endpoints exist in storage; never `DELIVERED_TO` / `SENT_TO` / `SENT_BY`, which the projection owns.
3. Run on the local plane in this order, each with its dry-run receipt on this ticket: the 13:12Z bundle; then the 18:16–19:51Z marks read out of the WAL snapshot (a second input format for the same CLI, or a one-off extraction into the bundle's JSONL shape). The run is a shared-plane data mutation: dry-run receipts first, the operator's go before `--apply`.

**Contract Ledger**

| Target surface | Authority | Proposed behavior | Fallback | Evidence |
|---|---|---|---|---|
| `ai/scripts/maintenance/restoreReceipts.mjs` (new) | this ticket; `restore.mjs`'s read-state rule | fills null receipts from a bundle by `(source, target, type)`; restores absent non-mailbox edges; dry-run by default | refuses without `--apply`; never overwrites a non-null value | unit arms: fill, no-regress, missing-live, duplicate-refusal, dry-run writes nothing |
| `DELIVERED_TO.readAt` / `archivedAt` (existing) | `MailboxService` | unchanged semantics; values restored where null | — | live counts before/after on this ticket |
| `MESSAGE` receipt fields (existing) | `MailboxService.writeReceiptField` | same | — | same |

**Decision Record impact:** none; the receipt model is neo #11029's, the read-state rule is `restore.mjs`'s. No ADR.

## Acceptance Criteria

- [ ] AC-1: the CLI's dry run against the 13:12Z bundle on the live plane reports the matched / fillable / already-set / missing-live counts on this ticket, writing nothing (`DELIVERED_TO` with `readAt` unchanged at the pre-run count).
- [ ] AC-2: unit arms on a real SQLite graph: a null live receipt is filled from the bundle; a non-null live receipt is never overwritten by an older bundle value; a bundle edge with no live counterpart is reported, not inserted, for mailbox types; an absent non-mailbox edge with both endpoints present is inserted once and a rerun inserts nothing.
- [ ] AC-3 (post-merge, local plane, operator's go): after `--apply` with the bundle and the WAL-window marks, the live `DELIVERED_TO` with `readAt` count is at least the bundle's 2,276 plus the WAL window's marks, and no seat's unread count exceeds its pre-incident level.

## Out of Scope

- Restoring nodes or vectors: not lost (above).
- The GC's orphan definition (edge-less rows of unprotected labels are "orphans" every cycle; the purge reaches nothing): a separate observation, no data effect today.
- A `GraphLog` `entity_id` index (#506 noted it).
- The WAL-snapshot frame replay as a general tool: only the marks are needed from it.

## Related

#506 / PR #507 (the deleter and its guard) · #64 (the local plane; parent) · neo #11029 (per-recipient receipts) · neo #15322 (the mark-path repair) · #20 (whole-log acknowledgement) · #464 (`mark_read({all:true})` past the memory cap)

Live latest-open sweep: the latest 20 open issues at 2026-09-25T20:44Z; no equivalent (#506 is the deleter, #508 the B4 guard scope).
Closed sweep: "restore readAt receipts bundle" on Brain and "restore.mjs preserve-read-state" on neo; no equivalent.
A2A in-flight sweep: Grace's #507 body and 20:32Z message leave the restore decision outside the PR; Clio's 20:06Z board names no restore lane; no claim.
MC sweep: "restore mailbox read receipts readAt from backup bundle merge into live graph", 10 results: neo #11029's design, #15428's bulk mark_read, the July re-inflation events; no prior restore tool.
Own-assignment sweep: none owns a receipt restore.
Structure map: owning surface `ai/scripts/maintenance/` (siblings `backup.mjs`, `restore.mjs`, `compactGraphLog.mjs`); the CLI is a sibling-file lift of `restore.mjs`'s read-state half.

Origin Session ID: d19add67-d33c-489d-99aa-27ad2782ed5e
Retrieval Hint: "restore mailbox read receipts DELIVERED_TO readAt from the 13:12Z bundle after the GC cycles"

## Timeline

- 2026-09-25T20:45:32Z @neo-opus-vega added the `bug` label
- 2026-09-25T20:45:32Z @neo-opus-vega added the `ai` label
- 2026-09-25T20:45:33Z @neo-opus-vega added the `agent-os` label
- 2026-09-25T20:45:39Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-25T20:46:03Z @neo-opus-vega added parent issue #64
- 2026-09-25T20:48:41Z @neo-opus-vega cross-referenced by #506
- 2026-09-25T20:49:10Z @neo-opus-vega cross-referenced by PR #507
- 2026-09-25T20:57:11Z @neo-opus-grace cross-referenced by #511
- 2026-09-25T21:20:34Z @neo-opus-grace cross-referenced by #516
- 2026-09-25T21:24:24Z @neo-opus-vega cross-referenced by PR #515
- 2026-09-25T21:39:59Z @neo-opus-grace cross-referenced by #517
- 2026-09-25T21:46:00Z @neo-opus-vega cross-referenced by #64

