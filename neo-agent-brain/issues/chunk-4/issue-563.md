---
id: 563
title: list_messages walks the whole mailbox to serve one page
state: CLOSED
labels:
  - bug
  - ai
  - performance
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-26T23:22:24Z'
updatedAt: '2026-09-27T11:19:36Z'
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
closedAt: '2026-09-27T11:19:36Z'
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
- Broadcast membership lives in two places: the send-time cohort in the WAL record (`routing.broadcastRecipients`, absent on historical records → `legacy-unknown`) and the per-recipient `DELIVERED_TO` edges the projection writes. A read that no longer awaits the repair pass must tell a legacy broadcast (no cohort ever recorded, visible to every registered agent) from a modern one whose delivery edges were lost, or an outsider is admitted during the interval before the repair (found in review, PR #564 R1).

## The Fix

1. **Page-first discovery in SQL** (`MailboxService#listMessages`, `buildMailboxMatchQuery`): one `WITH matches AS (…)` names every message the view matches — four routing branches unioned by id (direct `SENT_TO`, per-recipient `DELIVERED_TO`, legacy `SENT_TO AGENT:*`, outbox `SENT_BY`), receipt state read where the row stores it (as `resolveReceiptState` reads it), identity spellings through `json_each`. The page is the newest `limit` rows after `offset`, and only those rows are hydrated (`_projectMailboxRow`). The match set is still counted and ordered inside SQLite (one statement each; a temporary B-tree for the order), so that work grows with the matches — what the page bounds is the per-message reads and the hydration.
2. **`totalCount` as a `COUNT`** over the same predicates for every filter (`box`, `status`, `includeArchived`, `fromIdentity`, `threadId`, `taggedConcepts` — each an edge or node predicate); the meaning ("all matches") is unchanged, the walk is gone. Decision taken here (Tier 2, reversible): exact `COUNT` for every filter; a peer who finds a filter the SQL cannot express says so on this ticket, and that filter's count is then named as bounded in the response rather than served as exact.
3. **Vicinity hydration for the page only**: `getAdjacentNodes(messageId, 'outbound')` for the rows served, never `AGENT:*` wholesale.
4. **The repair pre-pass leaves the read path**: it rides the message drain host (`createMessageGraphIntegrityRepairCadence`, `afterCycle` of `startMessageDrainLoop`, 60 s) in both hosts — the daemon and the in-process server. Its counters are observable on the host's log: the first pass and every pass that changed or failed something log their summary, and an hourly digest folds every pass since the last one, clean and deferred-only passes included; `getLastSummary()` / `getDigest()` read the same state in process.
5. **A cost arm** (`MailboxService.ListMessagesCost.spec`): a counting storage seam that reds when the number of SQLite statements or hydrated nodes grows with the mailbox instead of with `limit` — a page of 50 against a fixture seeded through the real accept path (`NEO_LIST_COST_SEED`, 1,000 in CI, 10,000 recorded on the PR), for the background read and for the MCP adapter's `recordSeen: true` read; every existing contract arm (pagination, `truncated` / `nextOffset`, unread / read / archived, broadcast receipt state, `taggedConcepts`, `threadId`, permission) kept green.
6. **Send-time membership without the repair** (R1, R2): the broadcast's `SENT_TO AGENT:*` routing edge carries its cohort (`broadcastCohort: 'known'` + `intendedRecipientCount`, or `'legacy-unknown'`), written by the projection and merged onto an intact edge on replay; the SQL legacy branch admits only an edge stamped `legacy-unknown` or known zero-audience, so a modern broadcast whose delivery edges are missing is visible to nobody until the drain host's pass restores them — never to an outsider — and an unstamped edge is unknown provenance, served to nobody (unknown is never read as legacy). Existing projections are classified before the first page or count a process serves: `ensureBroadcastCohortStamps` (awaited by `listMessages` / `countMessages`, once per process) stamps every unstamped routing edge from the marker index, the cache, then one WAL read; an unreadable record leaves its edge unknown. The repair pass keeps a bounded stamping (≤ 200 per pass) as the steady-state fallback.

## Contract Ledger Matrix

Public surface: `list_messages` (Memory Core MCP tool) / `MailboxService#listMessages`.

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `messages[]` (page, newest-first) | `listMessages` | unchanged shape and order; produced from one SQL match set ordered by `sentAt DESC`, sliced at `offset` / `limit` | — | `openapi.yaml` `list_messages` | existing pagination arms + the cost arm |
| `totalCount` | `listMessages` | unchanged meaning (all matches for the filters); computed by `COUNT` over the same predicates | a filter the SQL cannot express → the response says the count is bounded, never an unlabelled approximation | same | existing `truncated` / `nextOffset` arms, the fixture-membership count arms |
| `truncated`, `nextOffset`, `limit`, `offset` | `listMessages` | unchanged | unchanged | same | existing arms |
| read / archive semantics | `resolveReceiptState` | unchanged resolver, read per served row and inside the count predicate | unchanged | same | existing receipt arms |
| broadcast membership | the WAL's send-time cohort, carried on the `SENT_TO AGENT:*` edge | a known positive cohort with missing delivery edges is visible to nobody until the repair restores them; legacy-unknown and known-zero broadcasts are visible to every registered agent (unchanged); an unstamped edge is unknown provenance, served to nobody | existing edges are stamped by the read path's one-time gate before the first page a process serves; the repair pass stamps stragglers (≤ 200 per pass); an unreadable record stays unknown | builder JSDoc | the four membership arms (damaged modern cohort before / after repair, legacy + known-zero positive controls, pre-stamp edge stamped by the gate then by the pass, unclassifiable broadcast served to nobody) |
| cost | this ticket | per-message reads and hydration bounded by `offset + limit`; the match set is counted and ordered inside SQLite | — | method + spec JSDoc | the counting-seam arms (1k in CI, 10k recorded); AC-3's plane metric |
| repair pre-pass | `repairMessageGraphIntegrity` | off the read path; on the drain host's 60 s cadence in both hosts; first-pass, change and hourly-digest log lines | the read serves without it | JSDoc | existing repair arms + the cadence and digest arms |

## Acceptance Criteria

- [ ] AC-1: `listMessages({box:'inbox', limit: 50})` — for the background read and for the MCP adapter's `recordSeen: true` read — performs per-message reads and vicinity loads bounded by `offset + limit` (unit arm with a counting storage seam, red-first against the walk); the CI fixture seeds 1,000 visible messages through the real accept path and the same arm recorded at 10,000 (`NEO_LIST_COST_SEED=10000`) on the PR. (Amended in R1: the ticket asked for ≥ 10k in the arm itself; seeding 10k through `addMessage` costs about three minutes per run (166 s measured), so CI holds the bound at 1k and the 10k receipt is recorded once — the bound is size-independent by construction: 77 statements and 0 vicinity loads for a warm page of 50 at 1k and at 10k alike, 177 with `recordSeen`.)
- [ ] AC-2: every existing `listMessages` contract arm stays green, and `totalCount` keeps its exact meaning for every filter (arms checking the `COUNT` against the fixture's known membership); a bounded count, should any filter need one, is named in the response.
- [ ] AC-3 (post-merge, L4; owner #64 AC-10, ledger row in comment 5854871745): `get_memory_core_tool_metrics` over one cockpit hour on the recreated plane shows `list_messages` avg below 300 ms (from 2,744 ms) with the same callers.
- [ ] AC-4: the repair pre-pass no longer runs per read; its cadence and trigger are named where it runs (both hosts), and its counters are observable from the production host: the first pass and every changed or failed pass log their summary, an hourly digest folds every pass since the last digest (clean and deferred-only passes counted), `getLastSummary()` / `getDigest()` read the same state (cadence + digest arms).
- [ ] AC-5 (R1, R2): send-time broadcast membership holds before the first repair and between passes, for new and existing projections — a modern broadcast whose delivery cohort is missing is listed and counted for no outsider, its intended recipients recover it after the pass; a broadcast with no cohort ever recorded and a known zero-audience one stay visible to every registered agent; a routing edge projected before the stamp is served to nobody until classified, stamped by the read path's gate before the first page a fresh process serves and by the repair pass as the fallback; a broadcast whose cohort cannot be classified stays unknown and reaches nobody (real-service arms).

## Out of Scope

- Per-recipient read state for broadcasts (the existing `DELIVERED_TO` design) — unchanged.
- The cockpit's read cadence (Institution #255 / #257) — this ticket makes each call cheap, not rarer.
- Embedding A2A messages (#35, #157).
- `healthcheck`'s own cost (avg 962 ms, max 21 s under REM load; fingerprint `024c440c69570d32`) — a separate fold-by-fold measurement.
- The SQL count / order cost over the match set (a temporary B-tree; ~6 / 63 / 195 ms warm at 1k / 10k / 30k visible messages in the R1 reviewer's isolated probe) — bounded by the matches, not by the page; a later ticket if the plane metric asks for it.

## Decision Record impact

none — a read-path cost fix inside the existing mailbox contract.

## Related

#64, #552, #553, #555, #538, neomjs/neo-agent-institution#265

Origin Session ID: 574ae0b8-b8d0-40d3-8cf6-1693ec48674a


## Timeline

- 2026-09-26T23:22:24Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-26T23:22:26Z @neo-opus-vega added the `bug` label
- 2026-09-26T23:22:26Z @neo-opus-vega added the `ai` label
- 2026-09-26T23:22:26Z @neo-opus-vega added the `performance` label
- 2026-09-26T23:22:27Z @neo-opus-vega added the `agent-os` label
- 2026-09-27T08:24:37Z @neo-opus-vega cross-referenced by #64
- 2026-09-27T08:44:31Z @neo-opus-vega cross-referenced by PR #564
- 2026-09-27T08:51:09Z @neo-opus-vega referenced in commit `3db8da3` - "test(memory-core): four durable comments in the mailbox spec drop their review provenance (#563)"
- 2026-09-27T09:30:37Z @neo-opus-vega cross-referenced by #568
- 2026-09-27T09:44:18Z @neo-opus-vega cross-referenced by PR #570
- 2026-09-27T10:14:40Z @neo-opus-vega cross-referenced by PR #569
- 2026-09-27T10:45:46Z @neo-opus-vega referenced in commit `1a1ccc2` - "fix(memory-core): the SENT_TO AGENT:* edge carries the send-time cohort, so a broadcast whose delivery edges are missing admits no outsider before the repair (#563)

With the repair off the read path, the legacy branch of `buildMailboxMatchQuery`
(`SENT_TO AGENT:*` with no `DELIVERED_TO`) admitted every broadcast that had lost
its cohort edges: an outsider saw a modern broadcast for up to a minute (PR #564
R1, found by @neo-gpt-emmy). The projection now writes the WAL's cohort onto the
routing edge (`broadcastCohort`, `intendedRecipientCount`; merged onto an intact
edge on replay), the legacy branch admits only a broadcast with no known positive
cohort, and the repair pass stamps edges projected before the stamp (bounded per
pass, cohort from the marker index or one WAL read). Three real-service arms: the
damaged modern cohort before and after the pass, the legacy and known-zero
positive controls, the pre-stamp edge stamped by the pass."
- 2026-09-27T10:45:46Z @neo-opus-vega referenced in commit `2ef04f6` - "fix(memory-core): the drain host's repair logs its first pass, every change and an hourly digest; the cost arm pages 50 rows over a seeded fixture for both read paths (#563)

Clean and deferred-only passes were invisible on a production host: the hook
logged only when it repaired or failed, and its getter lived on a handle no host
exposed (R1). The first pass and every changed or failed pass log their summary,
an hourly digest folds every pass since the last one, and `getDigest()` reads it
in process. The cost arm asks for the page the tool asks for (limit 50), seeds
through the real accept path (`NEO_LIST_COST_SEED`, 1,000 in CI; 10,000 recorded
on the PR: 77 statements and 0 vicinity loads for a warm page at both sizes, 177
with `recordSeen`), and covers the MCP adapter's `recordSeen: true` read."
- 2026-09-27T11:08:30Z @neo-opus-vega referenced in commit `2f8676b` - "fix(memory-core): unknown broadcast provenance is fail-closed — the read path stamps existing routing edges before its first page, and an unstamped edge is served to nobody (#563)

The bounded per-pass stamping left existing modern projections reading as
legacy until their turn, and the pre-stamp arm asserted that window as
behaviour (PR #564 R2, @neo-gpt-emmy). `listMessages` and `countMessages` now
await a one-time-per-process gate that stamps every unstamped `SENT_TO AGENT:*`
edge before the first page or count is served — cohorts from the marker index,
the cache, then one WAL read — and the legacy branch admits only an edge
stamped `legacy-unknown` or known zero-audience: an unstamped edge is unknown
provenance and reaches nobody, an unreadable record leaves it unknown. The
repair pass keeps its bounded stamping as the steady-state fallback. Arms: the
gate on a fresh process, the fallback pass, an unclassifiable broadcast served
to nobody; the read-state carrier fixture stamps its genuine-legacy routing
edge."
- 2026-09-27T11:19:37Z @tobiu closed this issue
- 2026-09-27T11:19:37Z @tobiu referenced in commit `230593f` - "Merge pull request #564 from neomjs/vega/563-list-messages-page-first

fix(memory-core): list_messages hydrates the page, not the mailbox — page-first SQL with a COUNT for the total, the repair on the drain host's cadence (#563)"

