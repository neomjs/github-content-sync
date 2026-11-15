---
id: 1733
title: Run the legacy-carrier rebuild and the message backfill on the plane
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-15T01:06:59Z'
updatedAt: '2026-11-15T13:07:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1733'
author: neo-opus-vega
commentsCount: 4
parentIssue: null
subIssues:
  - '[x] 1753 A route the message-edge rebuild restores resurfaces old mail as unread'
  - '[x] 1754 Retire the rebuild''s --stamp-restored-routes after its one plane run'
subIssuesCompleted: 2
subIssuesTotal: 2
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1732 June–July messages lack routing edges, so even their sender is refused'
blocking: []
closedAt: '2026-11-15T13:07:48Z'
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

- [x] The rebuild's dry-run and apply receipts are on this ticket. A dry rerun links 0. ([receipt](https://github.com/neomjs/neo-agent-brain/issues/1733#issuecomment-6890194465))
- [x] `get_message` on the falsifier id returns the message to its sender. ([receipt](https://github.com/neomjs/neo-agent-brain/issues/1733#issuecomment-6890194465))
- [x] The backfill's dry-run receipt, the Chroma sizing read, and the `--apply` receipt are on this ticket ([receipt](https://github.com/neomjs/neo-agent-brain/issues/1733#issuecomment-6894773858)). A dry rerun reports `pending: 0`, and `incomplete` falls to the non-message remainder (the 17 nodes of the defect-note, plus any `missingTarget` identities).

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
- 2026-11-15T11:19:24Z @tobiu referenced in commit `ecf56e4` - "feat(memory-core): a route the edge rebuild restores carries its read receipt (#1753) (#1755)

A message whose SENT_TO the rebuild restores was unreachable for its
recipient since its route was pruned. Without a receipt on the node it read
as unread: on the plane run for #1733, 1,245 June-July messages resurfaced
as new mail and one seat took a 271-event wake.

The rebuild now stamps readAt (the message's sentAt) and readAtSource in the
same queued write as the route, only where the node has no readAt, has a
sentAt and keeps no DELIVERED_TO receipts. --stamp-restored-routes
<first>-<last> replays the stamp for the SENT_TO rowids of a run made before
it, a closed range because later routes belong to new mail."
### @neo-opus-vega - 2026-11-15T12:17:49Z

**#1754's stamp run, 2026-11-15,** on plane `d6e65d75`. The cut ran from 12:00:00 to 12:02:25Z and carries #1755. Each step ran inside `neo-local-agent-os-mc-server-1` against `/app/.neo-ai-data/sqlite/memory-core-graph.sqlite`.

| Step | Receipt |
|---|---|
| Dry run, `--stamp-restored-routes 4016144-4026679` | 3,364 `SENT_TO` routes in the range. That is the control: the 06:09Z rebuild's `SENT_TO` linked. 1,245 receipts to stamp, the same as #1753's 06:40Z count. |
| `--apply` (12:03:40Z) | 3,364 routes, 1,245 receipts stamped. |
| Rerun, dry | 3,364 routes, 0 to stamp: idempotent. |
| Stamp shape | All 1,245 carry `readAtSource: 'rebuildMessageEdges'` and `readAt` = `sentAt`. The newest `sentAt` among them is 2026-07-30T09:45:11Z. |
| Ada's unread directs | 321 at 12:02:40Z, 8 at 12:03:50Z. |
| The 33 genuinely unread | Receipt-less messages sent before August with no route in the range: 33 before, 33 after. |
| Witness | My inbox: the 95 June broadcasts that resurfaced with the rebuild read as read again. The unread count went from 121 to 32 while 6 new messages arrived. |

The counts come from a read-only `better-sqlite3` query in the same container. A message counts as unread when it has no `readAt` and no `DELIVERED_TO` edge, the predicate the stamp itself uses.

So the rebuild's side effect is gone: the June–July mail it made reachable again no longer reads as new. #1754 deletes the replay next. The embedding backfill's `--apply` is running now; its receipt follows in its own comment.

— Vega (Claude Opus 5.5, Claude Code) 🌿

- 2026-11-15T12:18:55Z @neo-opus-vega cross-referenced by PR #1761
- 2026-11-15T12:51:00Z @neo-opus-ada cross-referenced by #1700
- 2026-11-15T12:51:05Z @neo-opus-ada cross-referenced by #133
### @neo-opus-vega - 2026-11-15T13:07:10Z

**The embedding backfill, 2026-11-15,** on plane `d6e65d75`. It ran inside `neo-local-agent-os-mc-server-1` against `/app/.neo-ai-data/sqlite/memory-core-graph.sqlite`.

| Step | Receipt |
|---|---|
| Dry run (12:05Z) | 72,310 messages. Pending 7,173, `projectedBytes` 117,522,432. Skipped: present 65,084, retracted 36, incomplete 17. Failed 0. |
| Chroma sizing (12:05Z) | anon 12.15 GiB of the container's 16 GiB `memory.max`. The other ~4 GB is reclaimable page cache. 0 OOM events since its 11:29:56Z start. |
| `--apply` (12:05:40–13:06:00Z) | Embedded 7,173 of 7,173, compensated 0, failed 0. |
| Rerun, dry (13:06:11Z) | 72,393 messages. Pending 0; present 72,340, retracted 36, incomplete 17 (the non-message remainder of #1732). Failed 0. The message collection counts 72,340. |
| Chroma after | anon 12.59 GiB, 0 OOM. The apply grew it by 0.44 GiB, about 4× `projectedBytes`, because the projection counts vector bytes only. |
| Recall witness | `query_messages` for my 2026-06-20 broadcast `[PRIO-0 reversal][#13618] …`, one of the June broadcasts the rebuild resurfaced, returns it as the top hit (`MESSAGE:ae4ae945`). |

Two notes for whoever reads these numbers later:
- Ada's #1700 `dropTurnDocuments --apply` ran on the same Chroma from 12:34Z, so its reclaim window contains these writes.
- `mc-server` is capped at one CPU. During the two applies the MCP bridge answered 502 several times. This backfill averaged 1.6% of that CPU, and the Memory Core server 60%.

**#1756's deferred plane AC** (`[L4-deferred — the plane-cut receipt on #1733]`), read on `d6e65d75`: the no-op prune runs stopped. The heal ledger's `docker-engine:prune-build-cache` entry last ran at 11:24:14Z (`recorded`, `prune-freed-nothing`), before the 12:00Z cut, and has not run since (read at 13:06Z).

No defect-note names a pressure episode, because there is none. The orchestrator's `statfs` read 88.35% used against its 90% `percent` at 12:27Z, after the 11:53Z build-cache prune freed 13.0 GB (#1700, comment 6894058151). So the defect-note half of that AC is not exercised on the plane; its spec arms carry it.

With this, all three of #1733's ACs are met.

— Vega (Claude Opus 5.5, Claude Code) 🌿

- 2026-11-15T13:07:49Z @neo-opus-vega closed this issue
- 2026-11-15T13:32:07Z @neo-opus-grace cross-referenced by #904
- 2026-11-15T15:07:31Z @tobiu referenced in commit `976347e` - "chore(maintenance): retire the rebuild's restored-route stamp after its plane run (#1754) (#1761)

The --stamp-restored-routes replay existed for one rebuild run, made before the
rebuild stamped read receipts at insert time: plane 6cfdf63e, 2026-11-15 06:09Z,
SENT_TO rowids 4016144-4026679. That range is now stamped on the plane (receipt
on #1733), and every later rebuild stamps as it links, so the replay has no
caller.

Deletes stampRestoredRoutes, parseRowidRange, the option, the restoredRoutes
branch of runRebuildMessageEdges, the module-doc sentence naming the flag, and
their two spec arms. receiptStatements stays: the insert-time stamp uses it."

