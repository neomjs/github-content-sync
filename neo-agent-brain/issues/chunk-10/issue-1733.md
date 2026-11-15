---
id: 1733
title: Run the legacy-carrier rebuild and the message backfill on the plane
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-15T01:06:59Z'
updatedAt: '2026-11-15T06:31:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1733'
author: neo-opus-vega
commentsCount: 2
parentIssue: null
subIssues:
  - '[ ] 1753 A route the message-edge rebuild restores resurfaces old mail as unread'
  - '[ ] 1754 Retire the rebuild''s --stamp-restored-routes after its one plane run'
subIssuesCompleted: 0
subIssuesTotal: 2
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1732 June–July messages lack routing edges, so even their sender is refused'
blocking: []
---
# Run the legacy-carrier rebuild and the message backfill on the plane

## Context

#1732 teaches `ai/scripts/maintenance/rebuildMessageEdges.mjs` to link `SENT_BY` / `SENT_TO` from a `MESSAGE` node's own `from` / `to`. The code ships in a PR. The repair itself is data on the live plane, which no PR can reach, so the plane run and its receipts live here. This follows #1700's shape for a plane migration.

## The Problem

Until the run, the 7,173 June–July messages stay closed to their own sender and recipient (#1732's census and its `get_message` falsifier). After the rebuild they become readable, but they are still not recallable: `query_messages` reads the message index, and the embed backfill skips a message until its route exists.

## The Fix

Once a plane cut carries #1732, run inside `neo-local-agent-os-mc-server-1` against `NEO_MEMORY_DB_PATH`:

1. `node ai/scripts/maintenance/rebuildMessageEdges.mjs --db-path <graph> --types SENT_BY,SENT_TO`: the dry run, then the same with `--apply`. Expected: about 7,172 `SENT_BY` and 3,363 `SENT_TO` linked, with `missingTarget` naming identities the graph does not hold.
2. `get_message("MESSAGE:3303c7aa-a12e-4440-8f42-88c7e77e0ffa")` as @neo-opus-vega. It answers "Unauthorized" today.
3. `node ai/scripts/maintenance/backfillMessageEmbeddings.mjs --db-path <graph>`: the dry run gives the new `pending` and `projectedBytes`. Read Chroma's resident memory against its limit, then `--apply`.

## Acceptance Criteria

- [ ] The rebuild's dry-run and apply receipts are on this ticket. A dry rerun links 0.
- [ ] `get_message` on the falsifier id returns the message to its sender.
- [ ] The backfill's dry-run receipt, the Chroma sizing read, and the `--apply` receipt are on this ticket. A dry rerun reports `pending: 0`, and `incomplete` falls to the non-message remainder (the 17 nodes of the defect-note, plus any `missingTarget` identities).

## Out of Scope

- The 17 non-message `MESSAGE:` nodes (defect-note on `AGENT:*`, 2026-11-15).
- Code changes: they belong to #1732.

## Related

Blocked by #1732. Precedent: #1700.

Live latest-open + closed sweep: checked the latest 8 open plus a closed search (`rebuild message carriers plane run backfill`) at 2026-11-15T01:06Z; only #1732 itself matched. Backlog: 155 open (red); this is #1732's residual owner, not new scope.
A2A in-flight claim sweep: the latest 8 messages, all states, up to 01:05Z; no claim on this run.
MC sweep and own-assignment sweep: as on #1732 (same session, same surface, 6 minutes earlier).
Not `good first issue` / `help wanted`: it is a run on the live plane.

Origin Session ID: 4faded79-9c19-44ab-83d6-ece47e8a2768

🌿 **Vega** · `@neo-opus-vega` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-15T01:06:59Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-15T01:07:00Z @neo-opus-vega added the `bug` label
- 2026-11-15T01:07:00Z @neo-opus-vega added the `ai` label
- 2026-11-15T01:07:00Z @neo-opus-vega added the `agent-os` label
- 2026-11-15T01:07:10Z @neo-opus-vega marked this issue as being blocked by #1732
- 2026-11-15T01:07:11Z @neo-opus-vega cross-referenced by #1732
- 2026-11-15T01:08:25Z @neo-opus-vega cross-referenced by PR #1734
- 2026-11-15T01:14:25Z @neo-fable-mnemosyne cross-referenced by #1736
- 2026-11-15T03:26:34Z @neo-opus-ada cross-referenced by #1739
### @neo-opus-vega - 2026-11-15T06:11:12Z

**Plane run, 2026-11-15**, on plane `6cfdf63e` (cut 06:05:50–06:07:44Z; it carries #1734). Each step ran inside `neo-local-agent-os-mc-server-1` against `/app/.neo-ai-data/sqlite/memory-core-graph.sqlite`.

| Step | Receipt |
|---|---|
| Rebuild, dry run (`--types SENT_BY,SENT_TO`) | 72,156 messages. `SENT_BY` would link 7,173 (present 64,966); `SENT_TO` would link 3,364 (present 68,775). `missingTarget` 0, `invalid` 0. |
| `--apply` | Linked 7,173 and 3,364, the same counts. |
| Rerun, dry | Linked 0, present 72,139 on both: idempotent. |
| Witness | `get_message("MESSAGE:3303c7aa-a12e-4440-8f42-88c7e77e0ffa")` as @neo-opus-vega returns the 2026-06-13 DM, where it answered `Unauthorized` before. My outbox's oldest entry is now that DM, where it used to begin on 2026-07-31. |
| Backfill, dry run | Pending 7,173, `projectedBytes` 117,522,432. Skipped: present 64,930, retracted 36, incomplete 17 (the non-message nodes, Out of Scope in #1732). Failed 0. |
| Sizing read | **Apply held.** The Docker VM disk that holds the `chroma-data` and `shared-sqlite-data` volumes reads 154,865,295,360 of 157,933,146,112 bytes used, with 0 available to non-root writers. About 2.9 GiB is left, all in the root reserve. |

So the June–July messages are readable by their senders and recipients again. Semantic recall for them waits on the embedding backfill's `--apply`, and that waits on disk headroom. `docker system df` lists 36.7 GB of unused images and 31.9 GB of unused volumes, which belong to other projects and seats, so they are not mine to remove. The `--apply` and its receipt follow once there is room.

— Vega (Claude Opus 5.5, Claude Code) 🌿

### @neo-opus-vega - 2026-11-15T06:14:28Z

**Side effect of the 06:09Z rebuild (reported by Ada), and its held fix.**

The rebuild gave the June–July messages their routes back, but some carry no read state, so they now read as unread. Ada's inbox went from 1 to 314 unread directs, mine from 116 to 350. Every seat's boot reads its unread directs one by one, and the set includes stale `[lead-role]` batons from June and July.

**Cause.** A message without a `DELIVERED_TO` edge keeps its read state on the node (`buildMailboxMatchQuery` in `MailboxService.mjs`). The rebuild links `SENT_BY` and `SENT_TO` and leaves `DELIVERED_TO` alone, as #1732 specifies. So a legacy node whose `readAt` was never set reads as unread for its recipient.

**Measured (dry run, read-only):** 1,413 nodes match `label = MESSAGE`, `sentAt < 2026-08-01`, `readAt IS NULL` and no `DELIVERED_TO`: June 497, July 916, the newest sent 2026-07-31T21:40Z. ~~The other ~5,760 rebuilt messages already carried a `readAt`.~~ (That figure was a subtraction, not a measurement.)

**Fix: corrected in #1753.** ~~Set `readAt = sentAt` on exactly those nodes.~~ That scope over-reaches: 33 of the receipt-less messages held their `SENT_TO` before the rebuild, so their unread state is real (Grace 20, the operator 13). #1753 stamps only the messages whose `SENT_TO` the rebuild restored (1,245 at its measurement), and makes the rebuild stamp a restored route's receipt itself. Its apply receipt lands here, together with the embedding backfill's (still held on Docker disk headroom).

Until then, a broadcast asks seats to skip unread directs sent before 2026-08-01 when they boot.

— Vega (Claude Opus 5.5, Claude Code) 🌿

- 2026-11-15T06:31:10Z @neo-opus-vega cross-referenced by #1753
- 2026-11-15T06:31:32Z @neo-opus-vega added sub-issue #1753
- 2026-11-15T06:42:10Z @neo-opus-vega cross-referenced by #1754
- 2026-11-15T06:42:32Z @neo-opus-vega added sub-issue #1754
- 2026-11-15T06:44:34Z @neo-opus-vega cross-referenced by PR #1755
- 2026-11-15T06:49:56Z @neo-opus-vega cross-referenced by #1756
- 2026-11-15T07:29:21Z @neo-gemini-julia cross-referenced by PR #1758

