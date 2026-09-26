---
id: 555
title: 'who_is_online still scans every AGENT_MEMORY row per agent, and its trail read takes the label index'
state: CLOSED
labels:
  - bug
  - ai
  - performance
assignees:
  - neo-opus-vega
createdAt: '2026-09-26T20:04:06Z'
updatedAt: '2026-09-26T20:36:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/555'
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
closedAt: '2026-09-26T20:36:03Z'
---
# who_is_online still scans every AGENT_MEMORY row per agent, and its trail read takes the label index

## Context

Post-merge measurement of #553 (Brain dev `b0b85e8`, the local plane recreated 19:54Z, the cockpit's 15 s tick still running): `who_is_online` fell from 6,445 / 7,410 ms (min / avg) to **877 / 1,484 ms** over 23 calls in six minutes. #552's bar was 300 ms; the two reads below are the remainder, measured read-only on the live graph.

| Read | Now | Why |
|---|---|---|
| `_projectAgentLiveness` → activity recency, per agent (`WakeSubscriptionService.mjs:1200–1205`) | **107 ms per agent × 16 agents ≈ 1.7 s** | `SELECT MAX(json_extract(timestamp)) … WHERE label = 'AGENT_MEMORY' AND agentIdentity = ?` runs on `idx_nodes_label` and walks all 35,806 `AGENT_MEMORY` rows for every agent |
| `_readReviewLifecycleLoad` trail read (`:999–1005`) | 579–612 ms busy (325 without the predicate; 129 idle) | the `AND json_extract(data, '$.label') = 'MESSAGE'` predicate makes the planner choose `idx_nodes_label` (29.8k rows) over the partial `idx_nodes_message_sent_at` (12.2k rows); `EXPLAIN QUERY PLAN` shows `SEARCH Nodes USING INDEX idx_nodes_label` |

## The Problem

#553 added the label index and the partial sent-time index; the recency read gained the label index and lost nothing else, so it still parses the whole memory table once per agent, and the trail read's defensive label predicate steers the planner away from the index written for it. Under the cockpit's tick mc-server stays at ~87 % of a core (the tick's `list_messages` and `healthcheck` reads are defect-noted separately).

## The Architectural Reality

- `MESSAGE:`-prefixed ids are written only by `MailboxService`, so the label predicate on the trail read is redundant by construction; the partial index already carries the `id LIKE 'MESSAGE:%'` bound.
- The recency read is one `MAX` per agent over a two-column key that a partial expression index can serve as a seek: `(json_extract(data, '$.properties.agentIdentity'), json_extract(data, '$.properties.timestamp')) WHERE json_extract(data, '$.label') = 'AGENT_MEMORY'`. The query already names both expressions, so no SQL changes.
- Both indexes are declared beside #553's in `ai/graph/storage/SQLite.mjs` (`CREATE INDEX IF NOT EXISTS`); the memory table is the largest label on the plane, so this index is the biggest of the three (a few MB, built once).

## The Fix

1. `SQLite.mjs`: add the partial composite index on `AGENT_MEMORY` (`agentIdentity`, `timestamp`).
2. `WakeSubscriptionService#_readReviewLifecycleLoad`: drop the label predicate (or pin `INDEXED BY idx_nodes_message_sent_at`); the spec's planner arm asserts the partial index for the service's real query text.
3. A spec arm on the recency read: with the index present, `EXPLAIN QUERY PLAN` names it; the projected `lastSeen` per agent is unchanged (equivalence control).

## Acceptance Criteria

- [ ] AC-1: `who_is_online` min/avg in `get_memory_core_tool_metrics` under the cockpit's 15 s tick drops below 300 ms on the local plane (35.8k memory rows, 16 identities); before: 877 / 1,484 ms.
- [ ] AC-2: `EXPLAIN QUERY PLAN` of the service's trail query names `idx_nodes_message_sent_at`, and of the recency query names the new index (spec arms, red at dev).
- [ ] AC-3: the roster's `lastSeen` / presence bands for a seeded set of agents are identical before and after (equivalence control).

## Out of Scope

- `list_messages` and `healthcheck` costs under the tick (defect-noted 20:03Z; separate promotion).
- The cockpit's cadence (Institution #255 / PR #257).

## Related

#552 · #553 · #64 (the plane receipts) · Institution #257

Live latest-open sweep: checked the latest 20 open Brain issues at 20:03Z; no equivalent (#554 is the ticket-reference resolver). A2A claim sweep: the 19:5xZ–20:03Z mailbox carries no claim on this scope. Memory sweep: `query_raw_memories` degraded (mc-server saturated). Own-assignment sweep: none of my open Brain tickets covers it.

Origin Session ID: 27467eea-851e-486b-a0ca-55f744b67fdf
Retrieval Hint: "who_is_online AGENT_MEMORY recency MAX per agent partial composite index trail read label predicate planner"

## Timeline

- 2026-09-26T20:04:07Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-26T20:04:08Z @neo-opus-vega added the `bug` label
- 2026-09-26T20:04:08Z @neo-opus-vega added the `ai` label
- 2026-09-26T20:04:08Z @neo-opus-vega added the `performance` label
- 2026-09-26T20:05:18Z @neo-opus-vega cross-referenced by #64
- 2026-09-26T20:14:26Z @neo-opus-vega cross-referenced by PR #556
- 2026-09-26T20:29:58Z @neo-gpt-emmy cross-referenced by PR #257
- 2026-09-26T20:36:03Z @tobiu referenced in commit `61c1963` - "Merge pull request #556 from neomjs/vega/555-who-is-online-recency-index

fix(memory-core): who_is_online's recency read rides an AGENT_MEMORY index and its trail read keeps the partial sent-time index (#555)"
- 2026-09-26T20:36:03Z @tobiu closed this issue

