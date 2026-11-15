---
id: 1753
title: A route the message-edge rebuild restores resurfaces old mail as unread
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-15T06:31:09Z'
updatedAt: '2026-11-15T11:19:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1753'
author: neo-opus-vega
commentsCount: 0
parentIssue: 1733
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[x] 1754 Retire the rebuild''s --stamp-restored-routes after its one plane run'
closedAt: '2026-11-15T11:19:23Z'
---
# A route the message-edge rebuild restores resurfaces old mail as unread

Parent: #1733

## Context

#1733's route rebuild (`rebuildMessageEdges --types SENT_BY,SENT_TO --apply`, plane `6cfdf63e`, 2026-11-15 06:09Z) restored 3,364 `SENT_TO` routes to June–July messages (receipt: #1733 comment 6890194465). Each of those messages had been unreachable for its recipient since its route was pruned. A message without a `DELIVERED_TO` edge keeps its read receipt on the node, so where `readAt` was never set it now reads as unread: Ada's unread directs went from 1 to 314, and her seat took one wake carrying 271 message events.

## The Problem

Measured on the live graph (read-only `better-sqlite3` queries inside `neo-local-agent-os-mc-server-1`, plane `6cfdf63e`, 06:23–06:29Z): **1,278** MESSAGE nodes have `sentAt < 2026-08-01`, `readAt` null and no `DELIVERED_TO` edge.

- **1,245** hold a `SENT_TO` the rebuild inserted: 95 June broadcasts and 1,150 directs (Grace 534, Ada 313, Euclid 96, Mnemosyne 90, Clio 73, Phoebe 28, the operator 16). These are the side effect.
- **33** held their `SENT_TO` before the rebuild (Grace 20 from 07-31, the operator 13 from 07-24 to 07-30). Their recipients could read them all along, so their unread state is real.

The fix held on #1733 (comment 6890228327: stamp `readAt` on every receipt-less message sent before 2026-08-01) would mark those 33 read as well, 13 of them in the operator's mailbox. No node property separates the two groups (key sets compared); only the route's provenance does.

**Instrument receipt.** The rebuild's routes are the top block of pre-August `SENT_TO` rowids, 4,016,144 to 4,026,679: exactly 3,364 edges, the rebuild's own `SENT_TO linked` count. The next lower pre-August `SENT_TO` sits at rowid 3,574,156. Control: Ada's 313 resurfaced plus the 1 she held before equals her reported 314.

## The Architectural Reality

- `ai/scripts/maintenance/rebuildMessageEdges.mjs` links a route through `insertRoute` (`INSERT … WHERE NOT EXISTS` a route of that type). That statement is the one place that knows the message had no receiving route, and it writes no receipt.
- Receipts: a direct message, and a legacy broadcast without `DELIVERED_TO`, keeps `readAt` on the MESSAGE node. `MailboxService.writeReceiptField` writes one JSON path through `SQLite.setRecordProperty` (`json_set`). The `node_update` trigger logs every `Nodes` update to `GraphLog`, so the live service's cache follows a write made outside it.
- List pages and counts are SQL over storage (`buildMailboxMatchQuery`), so a stamped node leaves every unread count at once.
- Wakes: `SENT_TO_ME` matches edge rows only and is unread-gated on the node's `readAt` (`heartbeatPulseEvaluator.mjs`). A receipt stamp fires no wake, and a route stamped in the same transaction as its insert wakes nobody.

## The Fix

In `rebuildMessageEdges.mjs`:

1. When `insertRoute` links a `SENT_TO`, the same queued op stamps the node: `readAt` = its `sentAt`, `readAtSource: 'rebuildMessageEdges'`. The predicate lives in the `UPDATE`: `readAt` null, `sentAt` present, no `DELIVERED_TO` edge. The `SENT_TO` row counts `readStamped`; a dry run counts what it would stamp.
2. `--stamp-restored-routes <first>-<last>`: for a plane that ran the rebuild before (1), the same stamp on messages whose `SENT_TO` rowid lies in the inclusive range. A closed range, because routes inserted after the run belong to new mail.

`readAtSource` keeps the stamp auditable and reversible; `readAt = sentAt` keeps it from reading as recent activity.

## Decision Record impact

none

## Acceptance Criteria

- [ ] A restored `SENT_TO` stamps `readAt` and `readAtSource` on a receipt-less node in the same batch; a node with a `readAt`, without a `sentAt`, or with a `DELIVERED_TO` edge keeps its state (one spec arm each).
- [ ] A restored `SENT_BY` alone stamps nothing: its recipient could already read the message.
- [ ] The dry run writes nothing and counts `readStamped`; a rerun stamps 0.
- [ ] `--stamp-restored-routes` stamps exactly the receipt-less messages whose `SENT_TO` rowid lies in the range; messages routed below and above it stay untouched (spec).

## Post-Merge Validation

- [ ] The plane run of `--stamp-restored-routes 4016144-4026679` and its receipt on #1733 belong to #1754, which then deletes the replay.

## Out of Scope

- The 33 genuinely unread messages: their recipients read them or not.
- The embedding backfill's apply (#1733, held on Docker disk headroom).
- Per-recipient receipts for legacy broadcasts: the recipient snapshot is not on the node (#1732).

## Avoided Traps

- A `sentAt` cutoff alone: it stamps the 33 genuine ones.
- A rowid floor without a ceiling: it stamps mail that arrived after the run.
- `archivedAt` instead of `readAt`: it hides restored mail from every listing.
- A separate script run after each rebuild: a second step every operator must remember, and the provenance exists only inside the rebuild's insert.

## Related

#1732 (legacy carriers) · #1734 (the rebuild) · #1733 (the plane run, parent)

Live latest-open + closed sweep: latest 20 open + closed searches ("unread read state legacy messages", "rebuildMessageEdges readAt") at 2026-11-15T06:29Z, Out of Scope read on same-surface hits (#616, #563); no equivalent or prior exclusion. Backlog: 161 open (red); #1733 owns the surface, so this files as its sub.
A2A sweep: inbox, last 30 messages (all read states), no overlapping claim.
MC sweep: "old June July messages show as unread after routes came back, wake burst", 5 results, stale-wake incidents of other mechanisms, no prior decision.
Own-assignment sweep: 13 open, #1733 the only same-surface one (parent).
Structure map: `ai/scripts/maintenance` owns it; no new file.

Retrieval Hint: "rebuild restored SENT_TO routes resurface legacy mail as unread", "readAtSource rebuildMessageEdges"

## Timeline

- 2026-11-15T06:31:09Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-15T06:31:11Z @neo-opus-vega added the `bug` label
- 2026-11-15T06:31:11Z @neo-opus-vega added the `ai` label
- 2026-11-15T06:31:11Z @neo-opus-vega added the `agent-os` label
- 2026-11-15T06:31:32Z @neo-opus-vega added parent issue #1733
- 2026-11-15T06:31:49Z @neo-opus-vega cross-referenced by #1733
- 2026-11-15T06:42:10Z @neo-opus-vega cross-referenced by #1754
- 2026-11-15T06:42:34Z @neo-opus-vega marked this issue as blocking #1754
- 2026-11-15T06:44:34Z @neo-opus-vega cross-referenced by PR #1755
- 2026-11-15T11:19:23Z @tobiu referenced in commit `ecf56e4` - "feat(memory-core): a route the edge rebuild restores carries its read receipt (#1753) (#1755)

A message whose SENT_TO the rebuild restores was unreachable for its
recipient since its route was pruned. Without a receipt on the node it read
as unread: on the plane run for #1733, 1,245 June-July messages resurfaced
as new mail and one seat took a 271-event wake.

The rebuild now stamps readAt (the message's sentAt) and readAtSource in the
same queued write as the route, only where the node has no readAt, has a
sentAt and keeps no DELIVERED_TO receipts. --stamp-restored-routes
<first>-<last> replays the stamp for the SENT_TO rowids of a run made before
it, a closed range because later routes belong to new mail."
- 2026-11-15T11:19:24Z @tobiu closed this issue
- 2026-11-15T12:18:55Z @neo-opus-vega cross-referenced by PR #1761

