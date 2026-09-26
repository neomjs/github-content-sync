---
id: 537
title: 'The decay lock trusts a cached clock, and a message''s edges are not exempt'
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-26T07:34:27Z'
updatedAt: '2026-09-26T07:34:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/537'
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
# The decay lock trusts a cached clock, and a message's edges are not exempt

## Context

Operator escalation, 2026-09-26: REM processing prunes too many edges, and mail items must be immune to Hebbian decay. The read-status loss itself is #506 (the GC severed live mailbox edges; #507 stopped it, #509 restores the receipts). This ticket is the other half: the ambient decay ran far more often than its 24-hour lock allows, and a message's own edges decay and prune like scent.

**Observed** (the plane's own log, `mc-server-2026-09-2{4,5}.log`, read on 2026-09-26):

| Day | `Running ambient topology decay` | Pruned |
|---|---|---|
| 09-24 | 17:02, 17:10, 17:19, 17:28, 17:39 — five runs in 37 minutes | 139 + 25 + 43 + 34 + 72 |
| 09-25 | 18:00:43, 18:09:50 — two runs 9 minutes apart | 54 + 241 |

Every run multiplies every unprotected edge weight by 0.98 and deletes the ones below 0.2. The graph's half-life is designed as ~79 days; on those two days it ran at 5 and 2 ticks per day.

**Observed** on the live graph (read-only, 2026-09-26 07:1xZ): `IN_REPLY_TO` and `TAGGED_CONCEPT` edges of messages sent 09-20..09-24 carry weights of 0.98^8 to 0.98^22 (8–22 ticks in ≤6 days). Messages sent on 09-25 before 18:09Z sit at 0.96 (the two runs); after, at 1.0. 3,565 `MESSAGE` nodes carry `inReplyTo`; 609 `IN_REPLY_TO` edges exist. 206 carry `partOfThread`; 5 `PART_OF_THREAD` edges exist. The three carriers are unaffected: 3,377 `DELIVERED_TO` rows delivered 09-20..24 all read 1.0 (neo #15973 shields them since 2026-07-26).

## The Problem

`GraphService.decayGlobalTopology` (`ai/services/memory-core/GraphService.mjs:970`) reads its clock through `this.db.nodes.get('_SYSTEM_STATE')` — the RAM cache — and, on a miss, creates a blank clock and runs. Anything that removes or replaces the clock outside this process's RAM makes the lock lie.

**The loop as it ran (measured):** on the revisions deployed through 09-25 18:09Z, `getOrphanedNodes` never named `SYSTEM_CLOCK`, and `_SYSTEM_STATE` is edgeless. Once a process had run the decay, the clock was cached; the next cycle's orphan pass deleted it from storage (`removeNodes` reaches storage for cached nodes, #517), and that cycle's decay found no clock and ran again. Every cycle after the first decay in a process's lifetime ran the decay; a container recreate (cold cache) ended a burst, which is the pattern in the table. The filter dates from 2026-05-09 (neo #10996); the edge weights show it has been ticking ~4×/day for weeks. #515 (`deada39`) protected `SYSTEM_CLOCK` and #520's allowlist keeps it, so the trigger is closed on `dev` since 09-25 21:32Z. The read is still RAM-only.

**Probe on `dev` (real SQLite graph):** seed `_SYSTEM_STATE` with `lastDecayedAt = now − 48h`; advance the row in storage to `now − 1 min` (another process's run); call `decayGlobalTopology()` — it runs (a `RELATES_TO` edge goes 1 → 0.98). The same-process re-run and a true cold cache both skip correctly.

**The mailbox half:** `MailboxService` writes six optional edges per message from the sender's declared fields — `IN_REPLY_TO`, `PART_OF_THREAD`, `REFERENCES_TICKET`, `TAGGED_CONCEPT`, `ORIGINATES_IN`, `RELATED_SESSION` (`MailboxService.mjs:2872-2889`). They are records: a reply is a reply forever, and `list_messages({threadId, inReplyTo, relatedTickets, taggedConcepts})` walks them. Only the three carriers are in `PROTECTED_EDGE_TYPES`; a type allowlist cannot exempt a message's `TAGGED_CONCEPT` without exempting every memory's, which is scent.

## The Architectural Reality

- `PROTECTED_EDGE_TYPES` (`GraphService.mjs:79`) is the decay-and-prune exemption; its criterion is record-vs-scent, per type.
- `_SYSTEM_STATE` is a storage fact; the node cache is lazy, LRU-shaped and requester-scoped (`ai/graph/Database.mjs`, `RequestScopedVicinitySet`). A 24-hour lock must not depend on cache residency.
- Message ids are `MESSAGE:<uuid>` by construction (`MailboxService.mjs:935`, `:1669`, `:2771`), so "an edge whose source is a message" is `substr(source, 1, 8) = 'MESSAGE:'` in the same SQL the decay already runs.
- The decay and prune statements are bulk SQL followed by `syncCache()`; the exemption belongs in their `WHERE`, beside the type placeholders.

## The Fix

1. `decayGlobalTopology` reads `lastDecayedAt` from storage (`SELECT json_extract(data, '$.properties.lastDecayedAt') FROM Nodes WHERE id = '_SYSTEM_STATE'`) and drops the pre-run cache probe and blank upsert. The post-run `upsertGlobalNode` (which lazy-loads before merging) creates or advances the clock as today. The JSDoc names why: the lock is a storage fact, and the #506-era loop is the anchor.
2. Both statements gain `AND substr(source, 1, 8) <> 'MESSAGE:'`; the `PROTECTED_EDGE_TYPES` block gets the one-line rule: a message's own edges are the sender's record, never scent.
3. Three arms in `test/playwright/unit/ai/services/memory-core/GraphService.spec.mjs`: (a) an external clock advance in storage makes an unforced run skip — red on `dev`; (b) a message-sourced `IN_REPLY_TO` at 0.05 and `TAGGED_CONCEPT` at 1.0 survive a forced run unchanged while a non-message `RELATES_TO` at 0.05 is pruned — red on `dev`; (c) `getOrphanedNodes` never returns `_SYSTEM_STATE` — green on `dev`, pins #515/#520.

**Contract Ledger**

| Target surface | Authority | Proposed behavior | Fallback | Evidence |
|---|---|---|---|---|
| `GraphService.decayGlobalTopology` lock | this ticket; the 24h lock's own JSDoc | the clock is read from storage on every call | no clock row → create and run (unchanged) | arm (a) |
| `GraphService.decayGlobalTopology` decay + prune predicate | `PROTECTED_EDGE_TYPES` criterion (record vs scent) | edges sourced by a `MESSAGE` node are neither decayed nor pruned | — | arm (b) |
| `getOrphanedNodes` | #520's allowlist | `_SYSTEM_STATE` never collectable | — | arm (c) |

**Decision Record impact:** none. Aligned with neo #15973 (carrier shielding) and ADR 0006 §2.5 by construction; no ADR.

## Acceptance Criteria

- [ ] AC-1: arm (a) is red against `dev` and green at the head: with the storage clock advanced by another writer and a stale cached copy, an unforced `decayGlobalTopology()` writes nothing.
- [ ] AC-2: arm (b) is red against `dev` and green at the head: message-sourced edges keep their weight and survive the prune; the non-message control at 0.05 is pruned.
- [ ] AC-3: arm (c) is green: `_SYSTEM_STATE` is never in `getOrphanedNodes()`.
- [ ] AC-4 (post-merge, local plane): after the recreate on the merged head, the first due decay logs one `Running ambient topology decay`, and the following cycles log `Skipping global topology decay (Algorithmic Lock: …)`; a read-only probe before and after that run shows message-sourced edge weights unchanged. Receipt on #64.

## Out of Scope

- Restoring the deleted read receipts: #509.
- Rebuilding the pruned message edges from the `MESSAGE` node's own fields (`inReplyTo`, `partOfThread`, `relatedTickets`, `taggedConcepts`, `originSessionId`, `relatedSessions`): its own ticket, filed beside this one.
- Reading the clock through `getNodeRecord`: it lazy-loads, then returns the cached copy, so a cached clock is still the RAM copy.
- Making `decayGlobalTopology` async (only `RemDigestion` calls it, with `await`): unnecessary for a prepared statement.

## Avoided Traps

- Exempting by adding `IN_REPLY_TO` etc. to `PROTECTED_EDGE_TYPES`: `TAGGED_CONCEPT` is shared with memories, where it is the Hebbian substrate proper. The exemption is by source label, not by type.
- Guarding the clock only through the orphan allowlist (#520): that closes one deleter; the lock must survive any writer, including a restore, a second process, or a future collector.

## Related

#506 / #507 (the GC deleter and its guard) · #509 (receipt restore) · #511 / #515 (`SYSTEM_CLOCK` protected) · #516 / #520 (orphan allowlist) · #517 (`removeNodes` reaches storage only for cached nodes) · #64 (local plane) · neo #15973 (carriers shielded) · neo #12644 (`RESOLVES` protected)

Live latest-open sweep: the latest 20 open issues at 2026-09-26T07:3xZ; no equivalent (#87 is the unread-carry decay of mailbox artifact state, a different mechanism; #517 is the cache-miss deleter).
Closed sweep: "decay lock", "ambient decay message edges" on Brain; "decayGlobalTopology" on neo — #15973 and #12644 are the precedents, not duplicates.
A2A in-flight sweep: the 30 newest messages at 07:3xZ; Clio's morning board names this lane as mine; no competing claim.
MC sweep: "ambient decay 24-hour algorithmic lock _SYSTEM_STATE lastDecayedAt reset" and "message edges exempt from ambient decay": no prior decision.
Own-assignment sweep: #509 (restore), #493, #471 — none owns the guard.
Structure map: owning folder `ai/services/memory-core/` (`GraphService.mjs` and its spec); no new file.

Origin Session ID: e2fd8a01-5bfc-4d82-ad5b-5c889f4b6710
Retrieval Hint: "decay lock reads cached _SYSTEM_STATE, orphan pass deleted the clock, decay ran five times in 37 minutes, message edges exempt from decay"

## Timeline

- 2026-09-26T07:34:28Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-26T07:34:29Z @neo-opus-vega added the `bug` label
- 2026-09-26T07:34:29Z @neo-opus-vega added the `ai` label
- 2026-09-26T07:34:29Z @neo-opus-vega added the `agent-os` label
- 2026-09-26T07:36:55Z @neo-opus-vega cross-referenced by #538

