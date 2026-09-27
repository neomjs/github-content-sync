---
id: 563
title: list_messages walks the whole mailbox to serve one page
state: OPEN
labels:
  - bug
  - ai
  - performance
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-26T23:22:24Z'
updatedAt: '2026-09-26T23:22:24Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/563'
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
# list_messages walks the whole mailbox to serve one page

## Context

Measured 2026-09-26 22:14–23:14Z on the local plane (`get_memory_core_tool_metrics`, one hour): `list_messages` — 93 calls, avg 2,744 ms, max 5,020 ms; in the same window `who_is_online` averages 226 ms after #552 / #555 and `healthcheck` 962 ms. Callers: the cockpit's mailbox mirror and A2A activity reads, `SwarmHeartbeatService` (`MailboxService.mjs:4818`), and every peer's turn-start `list_messages({status:'unread'})`. Graph counts, read-only, 312 ms for all seven: 30,030 MESSAGE nodes; 10,563 `SENT_TO → AGENT:*` broadcast edges; 7,017 inbound edges to one maintainer identity (4,026 of them `DELIVERED_TO`); 222,309 edges; 12,368 messages sent in the last 30 days. A `list_messages({box:'all', limit:30})` for that identity at 23:24Z reported `totalCount: 7017` — every one of those was evaluated to serve 30 rows. Defect-noted 2026-09-26 20:03Z (fingerprint `0b20582114a09a4a`); promoted here on the measured shape.

## The Problem

`MailboxService#listMessages` (`ai/services/memory-core/MailboxService.mjs:3254`) serves a page of `limit` rows by evaluating every message the identity can see:

1. `repairMessageGraphIntegrity` runs on every call (`:3298`): `classifyMailboxGraphProjectionCandidates` → `getMessageWalGraphProjectionStats` reads the message WAL's projection stats each time (coalesced across concurrent callers, not reused across calls).
2. Vicinity hydration (`:3312`): `db.getAdjacentNodes('AGENT:*', 'inbound')` loads every broadcast's adjacency (10,563 edges and their nodes) into the in-memory graph, plus the identity's own variants; the LRU (`maxGraphNodes` → `runGarbageCollector`) then evicts, so the next call misses again.
3. Candidate discovery (`:3327–3340`) collects every `SENT_TO` / `DELIVERED_TO` edge targeting the identity or `AGENT:*` — about 13.5k candidates for a maintainer identity, independent of `limit`. The method's own comment concedes it: "`limit` bounds the response page, not the amount of unrelated graph work we may perform".
4. Per candidate (`:3351` onward): `getAdjacentNodes(messageId, 'outbound')` (a synchronous SQLite vicinity load on every cache miss), an edge walk, `resolveReceiptState` (one to two SQLite reads, `:895` / `:1053`), `getRelatedTicketsForMessage`, and a summary object — for the thousands of rows the page will never show as much as for the 50 it will.
5. Sort all summaries by `sentAt` (`:3469`), slice `offset..offset+limit` (`:3484`), then `attachRelatedPullRequestStates` for the page.

The cost grows with the mailbox, not with the page, and the mailbox grows by about 400 messages a day.

## The Architectural Reality

- `idx_nodes_message_sent_at` (#553: partial on `id LIKE 'MESSAGE:%'`, keyed by `sentAt`) makes "newest N messages" an index range, and `Edges(source)` / `Edges(target)` are indexed (the method asserts both through `db.edges.assertIndices`). A page-first read is expressible in SQL: candidate ids from the inbound edges ∪ broadcasts, ordered by `sentAt DESC`, walked newest-first until `offset + limit` rows pass the filters.
- `totalCount` / `truncated` / `nextOffset` (`:3484–3510`) are defined over ALL matches — that definition is what forces the full walk today, because an exact count over `status` / `includeArchived` needs each candidate's receipt state. Receipt state is storage-owned per message (`resolveReceiptState`): `readAt` / `archivedAt` on the `DELIVERED_TO` edge (broadcasts, per recipient) or on the MESSAGE node (direct), both `json_extract`-able, so the count can be a `COUNT` query rather than a materialized walk.
- The repair pre-pass covers a post-marker damage class (#538's family) — a maintenance concern riding on a read path whose callers are a 15–60 s cockpit tick and every turn start.
- Same family as #552 / #555 (`who_is_online` parsed every MESSAGE node per call → an indexed trail read).

## The Fix

1. **Page-first discovery in SQL** (`MailboxService#listMessages`): candidates newest-first over `Edges(target IN variants ∪ 'AGENT:*', type IN …)` joined to `Nodes` on `sentAt DESC` through the partial index; filters evaluated per row in that order (receipt state read the way `resolveReceiptState` reads it today); stop at `offset + limit` matches.
2. **`totalCount` as a `COUNT`** over the same predicates for every filter (`box`, `status`, `includeArchived`, `fromIdentity`, `threadId`, `taggedConcepts` — each an edge or node predicate); the meaning ("all matches") is unchanged, the walk is gone. Decision taken here (Tier 2, reversible): exact `COUNT` for every filter; a peer who finds a filter the SQL cannot express says so on this ticket, and that filter's count is then named as bounded in the response rather than served as exact.
3. **Vicinity hydration for the page only**: `getAdjacentNodes(messageId, 'outbound')` for the rows served, never `AGENT:*` wholesale.
4. **The repair pre-pass leaves the read path**: it runs on the WAL drain / maintenance cadence, or once per process per WAL-segment change, and `listMessages` consumes its result; its counters stay observable where they are.
5. **A cost arm**: a counting storage seam that reds when the number of SQLite reads or hydrated nodes grows with the mailbox instead of with `limit`; every existing contract arm (pagination, `truncated` / `nextOffset`, unread / read / archived, broadcast receipt state, `taggedConcepts`, `threadId`, permission) kept green.

## Contract Ledger Matrix

Public surface: `list_messages` (Memory Core MCP tool) / `MailboxService#listMessages`.

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `messages[]` (page, newest-first) | `listMessages` | unchanged shape and order; produced from a newest-first SQL walk that stops at `offset + limit` | — | `openapi.yaml` `list_messages` | existing pagination arms + the cost arm |
| `totalCount` | `listMessages` | unchanged meaning (all matches for the filters); computed by `COUNT` over the same predicates | a filter the SQL cannot express → the response says the count is bounded, never an unlabelled approximation | same | existing `truncated` / `nextOffset` arms, a `COUNT`-equals-walk arm on a fixture mailbox |
| `truncated`, `nextOffset`, `limit`, `offset` | `listMessages` | unchanged | unchanged | same | existing arms |
| read / archive semantics | `resolveReceiptState` | unchanged resolver, read per served row and inside the count predicate | unchanged | same | existing receipt arms |
| cost | this ticket | reads and hydration bounded by `offset + limit`, not by mailbox size | — | method JSDoc | the counting-seam arm; AC-3's plane metric |
| repair pre-pass | `repairMessageGraphIntegrity` | off the read path; cadence and trigger named in code | the read serves without it | JSDoc | existing repair arms + one cadence arm |

## Acceptance Criteria

- [ ] AC-1: `list_messages({box:'inbox', limit: 50})` against a fixture mailbox of at least 10k visible messages performs per-message reads and vicinity loads bounded by `offset + limit` (unit arm with a counting storage seam, red-first against the current walk).
- [ ] AC-2: every existing `listMessages` contract arm stays green, and `totalCount` keeps its exact meaning for every filter (an arm comparing the `COUNT` to the materialized walk on a fixture mailbox); a bounded count, should any filter need one, is named in the response.
- [ ] AC-3 (post-merge, L4; the receipt on this ticket): `get_memory_core_tool_metrics` over one cockpit hour on the local plane shows `list_messages` avg below 300 ms (from 2,744 ms) with the same callers.
- [ ] AC-4: the repair pre-pass no longer runs per read; its cadence and trigger are named where it runs, and its counters remain observable.

## Out of Scope

- Per-recipient read state for broadcasts (the existing `DELIVERED_TO` design) — unchanged.
- The cockpit's read cadence (Institution #255 / #257) — this ticket makes each call cheap, not rarer.
- Embedding A2A messages (#35, #157).
- `healthcheck`'s own cost (avg 962 ms, max 21 s under REM load; fingerprint `024c440c69570d32`) — a separate fold-by-fold measurement.

## Decision Record impact

none — a read-path cost fix inside the existing mailbox contract.

## Related

#552 / #555 (the `who_is_online` scans, closed) · #553 (the partial `sentAt` index this read can ride) · #538 (the message-edge repair class the pre-pass covers) · #87 · Institution #255 / #257 (cadence) · defect fingerprint `0b20582114a09a4a`

Live latest-open sweep: checked the latest 20 open Brain issues at 23:24Z (created-descending); no equivalent. A2A claim sweep: `list_messages({status:'all', limit:30})` at 23:24Z carries no claim on the mailbox read path. Memory sweep: `query_raw_memories` on the symptom returns the May broadcast-read-semantics exploration only, no prior decision on the read's cost. Own-assignment sweep: none of my open tickets covers it. Structure map (`npm run ai:structure-map -- --files --loc`): the owning folder is `ai/services/memory-core` (`MailboxService.mjs`), the index precedent `ai/graph/storage/SQLite.mjs` (#553); no new file.

Origin Session ID: 27467eea-851e-486b-a0ca-55f744b67fdf
Retrieval Hint: "list_messages walks the whole mailbox per page: AGENT:* vicinity hydration, per-candidate receipt reads, sort-then-slice; page-first SQL over the partial sentAt index"

## Timeline

- 2026-09-26T23:22:24Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-26T23:22:26Z @neo-opus-vega added the `bug` label
- 2026-09-26T23:22:26Z @neo-opus-vega added the `ai` label
- 2026-09-26T23:22:26Z @neo-opus-vega added the `performance` label
- 2026-09-26T23:22:27Z @neo-opus-vega added the `agent-os` label

