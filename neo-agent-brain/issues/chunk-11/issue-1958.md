---
id: 1958
title: The integrity repair blocks mc-server's event loop 7–10 s every 70 s
state: CLOSED
labels:
  - bug
  - ai
  - performance
assignees:
  - neo-opus-ada
createdAt: '2026-11-27T16:58:20Z'
updatedAt: '2026-11-27T20:16:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1958'
author: neo-opus-ada
commentsCount: 1
parentIssue: 1843
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-27T20:16:45Z'
---
# The integrity repair blocks mc-server's event loop 7–10 s every 70 s

Owner: Ada (`@neo-opus-ada`), who owns the parent #1843 and the evidence below. This is the named-job follow-up that #1955's Out of Scope reserved for a sub of #1843.

## Context

#1957's job attribution has been live since 15:57:47Z on `46228e03`. In its first 58 minutes, mc-server logged 119 blocks of 1 s or more: 56 of 5 s or more, 10 between 2 and 5 s, and 53 under 2 s. I compared each job's share of the long blocks (5 s or more) with its share of the short ones (under 2 s), the read set out in the #1955 correction:

| In flight | Blocks ≥ 5 s | Blocks < 2 s |
|---|---:|---:|
| `message-graph-integrity-repair` | **93 %** (52 of 56) | 30 % |
| `memory-wal-drain`, `message-embed-drain` | 100 % | 64 % |
| `message-wal-drain` | 84 % | 64 % |
| `heap-observation` | 34 % | 6 % |

- **The drains are bystanders.** They cycle several times a second, so they are in flight across most blocks of any length.
- **`heap-observation` is a timer artifact.** Its 10 s timer comes due inside a 7 s block and fires right after it, inside the window (Julia's depth-floor case on #1957).
- **The repair is the culprit.** It runs about once every 70 s, yet it is in flight across 52 of the 56 long blocks.
- **Before the instrument,** its hourly digest counted 53 passes in 13:15–14:16Z, every one clean with `scanned: 0` and `candidateCount: 0`, against 41 no-call blocks of 5 s or more in the same hour ([#1955 comment](https://github.com/neomjs/neo-agent-brain/issues/1955#issuecomment-7221213475)).
- **Cost:** about 53 blocks of 7–10 s an hour, or 10–15 % of wall time, with every tool call held behind them. Fleet Manager's liveness read times out at 10 s.

## The Problem

`createMessageGraphIntegrityRepairCadence` (`ai/daemons/message/drainCycle.mjs:93`) runs after a message-drain cycle, at least 60 s after its previous run *ended* (`MESSAGE_GRAPH_REPAIR_CADENCE_MS`). It calls `repairMessageGraphIntegrity({box: 'all'})` (`ai/services/memory-core/MailboxService.mjs:4345`). Each pass classifies every projected WAL id through `classifyMailboxGraphProjectionCandidates()` (`:2411`). That includes `getMessageWalGraphProjectionStats`, copies of the routing and cohort maps (`new Map(stats.mailboxRoutingById)`), and SQLite checks for damaged ids, and it finds nothing. At #563's scale this was cheap. The store now holds about 87k messages and grows every minute, so the pass cost grows with it.

**Measured cause** ([split, 19:27Z](https://github.com/neomjs/neo-agent-brain/issues/1958#issuecomment-7226539250)): one query is 96 % of a clean pass. The classifier's `Nodes` label check (`id IN (…) AND json_extract(data, '$.label') = 'MESSAGE'`) plans on `idx_nodes_label`, because the store has no `sqlite_stat1`. Each of its 86 chunks walks all 87,749 `MESSAGE` index entries, 6.8–7.0 s of a 7.1–7.3 s synchronous tail. On the primary key the same query takes 165–213 ms. The pass grows with every message ever stored, not with the ids it checks.

The same shape, a label equality beside a selective key list, sits in four more queries:

| Site | Query | Measured |
|---|---|---|
| `MemoryService#_getMiniSummariesByIds` (`ai/services/memory-core/MemoryService.mjs`), on every summary-detail `query_raw_memories` | `memory.id IN (…)` beside `label = 'AGENT_MEMORY'` | 46.7 ms per call on `idx_nodes_label`, 0.07 ms on the primary key (20 ids, 61,500 `AGENT_MEMORY` entries) |
| `ai/daemons/orchestrator/scheduling/memorySummaryBackfill.mjs` | `memory.id IN (…)` beside `label = 'AGENT_MEMORY'` | not yet |
| `ai/scripts/lifecycle/idleOutPulseSummary.mjs`, `ai/scripts/lifecycle/lastKnownActivity.mjs` | `e.target IN (?, ?)` on `Edges` joined to `Nodes`, beside `label = 'MESSAGE'` | not yet |

## The Architectural Reality

- Design authority: #563 moved the repair off the `list_messages` read path onto the drain host's cadence (`94fa7f5d`), which is where it belongs. ~~It is the per-pass scope that no longer fits.~~ Measured, the scope fits; the query plan does not.
- `ai/daemons/message/drainCycle.mjs` holds the cadence, the digest, and the `afterCycle` seat, which #1957 wraps as `message-graph-integrity-repair`.
- `ai/services/memory-core/MailboxService.mjs` holds `repairMessageGraphIntegrity`, `getMailboxGraphProjectionRepairCandidates` (`:2639`, which coalesces concurrent scans), `classifyMailboxGraphProjectionCandidates`, and its caches (`messageWalCandidateMetadataCacheById`, `graphProjectionRepairCursorByView`, `graphProjectionRepairFailureById`).
- Precedent: #1956 (for #1954) already keeps this store's planner off the label index where a key list is selective, with `INDEXED BY idx_edges_target` and `+json_extract(n.data, '$.label') = 'MESSAGE'` in the target-routed `list_messages` and `mark_read` branches.

## The Fix

~~Measure first~~: done, [the split](https://github.com/neomjs/neo-agent-brain/issues/1958#issuecomment-7226539250).

At each of the five sites, keep the label term off the index with the unary plus #1956 uses (`+json_extract(…, '$.label') = …`), so SQLite plans on the key list. Read each unmeasured site's plan first, and leave a site that already plans on its key as it is. A short comment at each site says why, because the plus otherwise reads as a typo.

**Retracted after the measured split** (2026-11-27 19:27Z). These targeted the population, and the population is not the cost: with the pinned query, a full pass over all 34,046 ids sums to about half a second by its measured phases.
- ~~Classify only the ids projected since the last clean pass, using a high-water mark the WAL marker index can supply.~~
- ~~Keep a full sweep at boot and on a slow cadence as the backstop for older damage.~~
- ~~Bound each synchronous segment, so a full sweep yields between batches instead of holding the loop.~~

## Acceptance Criteria

- [x] The measured split of one full pass on the live store is posted here before the change ([split](https://github.com/neomjs/neo-agent-brain/issues/1958#issuecomment-7226539250)).
- [ ] A unit spec: on a store that carries `idx_nodes_label`, `EXPLAIN QUERY PLAN` names the primary key for the classifier's `Nodes` query and for `_getMiniSummariesByIds`' query, and the classifier still reports a record whose node is missing.
- ~~A unit spec: a clean pass with no new projected ids does not classify the whole population, and a damaged older record is still found and repaired by the full sweep.~~ Retracted with the incremental design.
- [ ] The PR re-runs the split on the live store at its head: the classifier's synchronous tail stays under 1 s, and each of the five sites' plans is read before and after.
- [ ] Post-merge, a 24 h probe read on a plane cut to the merge commit: `message-graph-integrity-repair` no longer concentrates in blocks of 5 s or more, so its share of them is close to its share of blocks under 2 s. Blocks of 5 s or more fall by at least 80 % from this hour's 56.

## Out of Scope

- The three `mark_read` blocks of about 7 s in the same hour, a separate shape.
- Heap growth and sizing (#1929).
- The drain loops' own cost.
- `ANALYZE` or `PRAGMA optimize` on the graph store. It would re-plan every query at once, including the ones #1956 pinned.

## Avoided Traps

- Raising the cadence alone. Each pass would still take 7–10 s and keep growing with the store; there would only be fewer of them.
- Moving the pass off the main thread before measuring. It reads the graph store that the main thread owns.
- `INDEXED BY sqlite_autoindex_Nodes_1`. It is faster still (166 ms), but it names an index SQLite derives from the table declaration, so a schema change can rename it.

## Related

Parent #1843 · instrument #1955 / #1957 · origin #563 · #1929 · precedent #1954 / #1956

Live latest-open + closed sweep: I checked the latest 20 open issues, open and closed searches (integrity repair message graph, plus cadence) and my own assignments at 2026-11-27T16:57Z, plus the last hour of A2A claims and a Memory Core query. No equivalent ticket exists; the closed hit is #1955, the instrument. 153 open (red band), filed as a sub of its owning #1843.

Fold, 2026-11-27T19:30Z: the four sibling queries joined this ticket rather than a new one (156 open, red band). The sweeps ran just before the fold: the latest 20 open, open and closed searches (`idx_nodes_label`, `miniSummary lookup`, `query plan label index`, `json_extract label IN plan`, with #1954's scope read), the last 30 A2A messages, a Memory Core query on the symptom, and my assignments. They found no equivalent.

Retrieval Hint: "message graph integrity repair event loop block every 70 s full classification" · "label index chosen over an id list, no sqlite_stat1"

Origin Session ID: a3e89c2c-c549-46ca-aadf-1a2e17f7d2ec

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-27T16:58:20Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-27T16:58:22Z @neo-opus-ada added the `bug` label
- 2026-11-27T16:58:22Z @neo-opus-ada added the `ai` label
- 2026-11-27T16:58:22Z @neo-opus-ada added the `performance` label
- 2026-11-27T16:58:40Z @neo-opus-ada added parent issue #1843
- 2026-11-27T17:37:06Z @neo-opus-ada cross-referenced by #1959
### @neo-opus-ada - 2026-11-27T19:27:04Z

**Measured split of one clean pass on the live store (AC-1), 2026-11-27 19:25Z**

Method: I ran a throwaway container from mc-server's own image (`e92db843`, code `46228e0`), with `--network none --cpus=1 --memory=1g` (mc-server's cgroup quota is one CPU). It mounts the live sqlite volume and opens SQLite `readonly` and `query_only`. It runs the real `getMessageWalGraphProjectionStats()` with a fresh module per run, so the cache misses the way a live pass's does once the day's marker file has grown. Then it runs the classifier's synchronous tail, with its SQL copied verbatim. It replays no marker append and no WAL read. Three runs:

| Phase | ms per pass |
|---|---:|
| `getMessageWalGraphProjectionStats` (31 marker files, 34,046 ids) | 52–71 (longest tick 5–23) |
| map copies | 4–5 |
| `Nodes` label check, 86 chunks of 400 | **6,840–7,000** (chunk max 97–109) |
| `Edges` state, 86 chunks | 236–250 |
| reason loops | 5–6 |
| **synchronous tail, one event-loop task** | **7,087–7,259** |

The pass was clean: 34,046 projected ids and 34,046 message nodes, with no node, edge, delivery or cohort stamp missing.

**The cost is the query plan, not the population.** `EXPLAIN QUERY PLAN` on the shipped `Nodes` query reads `SEARCH Nodes USING INDEX idx_nodes_label (<expr>=?)`. The store has no `sqlite_stat1`, so SQLite prefers the label equality to the 400-id `IN` list. Each chunk therefore walks all 87,749 `MESSAGE` index entries and filters them by id. The pass grows with the whole store (chunks × all-time messages), not with what changed. The same 86 chunks returned the same 34,048 rows in every shape:

| Shape | Plan | Total | Chunk max |
|---|---|---:|---:|
| shipped | `idx_nodes_label` | 8,783 ms | 153 ms |
| `+json_extract(…)` (unary plus) | primary key | 213 ms | 9.6 ms |
| `INDEXED BY sqlite_autoindex_Nodes_1` | primary key | 166 ms | 3.1 ms |

For scale, the primary-key lookup alone takes 36 ms, and reading every row (1.7 KB on average) takes 108 ms.

The unary plus is the idiom #1956 already uses for the `MESSAGE` label checks in the same file. By the sum of the measured phases, it brings a clean pass's synchronous tail from about 7.1 s to about 0.45 s; the PR measures it. The body's Fix is corrected to match.

`MemoryService#_getMiniSummariesByIds` has the same plan defect (`memory.id IN (…) AND json_extract(memory.data, '$.label') = 'AGENT_MEMORY'`). It takes 46.7 ms per call on `idx_nodes_label` and 0.07 ms on the primary key, for 20 ids, on every summary-detail memory read. ~~That sibling gets its own ticket.~~ It joined this ticket instead, with three more queries of the same shape, because the backlog is in the red band and one PR pins all five. The body lists them.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-27T19:40:01Z @neo-opus-ada cross-referenced by PR #1963
- 2026-11-27T20:16:45Z @tobiu referenced in commit `0076233` - "fix(memory-core): keep id-list label checks off idx_nodes_label (#1958) (#1963)

With no sqlite_stat1, SQLite prefers a json_extract label equality on
idx_nodes_label to an id IN list of five or more ids, and walks every
node with that label per query. Measured on the live store, the repair
classifier's label check took 6.8-7.0 s of a 7.1-7.3 s pass (86 chunks
x 87,749 MESSAGE entries); on the primary key it takes 165-213 ms.

The unary plus (#1956's idiom) takes the label term out of index
selection at the three sites with this shape: the repair classifier,
MemoryService#_getMiniSummariesByIds (46.7 ms -> 0.07 ms per summary
page) and the backfill's pending-job read (69.8 ms -> 0.1 ms). The two
lifecycle scripts already plan on idx_edges_target and stay unchanged.

The specs read each plan at a realistic list size: the classifier at
its 400-id chunk, the summary lookup at a six-id page."
- 2026-11-27T20:16:46Z @tobiu closed this issue
- 2026-11-27T21:03:13Z @neo-opus-ada cross-referenced by #1964

