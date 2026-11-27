---
id: 1954
title: Mailbox reads block mc-server's event loop for 10 to 24 seconds
state: CLOSED
labels:
  - bug
  - ai
  - performance
assignees:
  - neo-gemini-julia
createdAt: '2026-11-27T11:50:28Z'
updatedAt: '2026-11-27T12:56:44Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1954'
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
closedAt: '2026-11-27T12:56:44Z'
---
# Mailbox reads block mc-server's event loop for 10 to 24 seconds

Unowned: a sub of #1843 that #1940's profile named. Ready for any maintainer; intake starts with the plan measurement below.

## Context

#1940's 40 h probe receipt ([comment](https://github.com/neomjs/neo-agent-brain/issues/1940#issuecomment-7217517914), plane `191aaaa7`, 11-25 19:48Z → 11-27 11:47Z) attributes 36 of 93 event-loop blocks of 10 s or more to a culprit call. 24 of those 36 are mailbox reads and writes:

| Blocks ≥ 10 s | Max | Blocked total | Shape |
|---:|---:|---:|---|
| 14 | 24.4 s | 220.8 s | `list_messages {box:'all', status:'all', limit:50}` |
| 4 | 14.5 s | 51.7 s | `mark_read {all}` (+ `includeUnseen*`) |
| 3 | 18.0 s | 43.1 s | `list_messages {to, box:'inbox', status:'all', limit:51}` |
| 3 | 13.7 s | 37.0 s | `list_messages`: `fromIdentity` + `box:'all'` · `counts` unread · inbox compact unread |

GC stays at or below 0.3 s on these blocks, so the time is synchronous SQLite work, not heap. While one of these runs, every seat's call waits: `query_recent_turns` reached 45 s and `who_is_online` 30 s as bystanders.

## The Problem

#563 made a page cheap but kept one cost out of scope: "the SQL count / order cost over the match set … a later ticket if the plane metric asks for it." The plane metric now asks for it.

## The Architectural Reality

- `buildMailboxMatches` builds every `box:'inbox'|'all'` read as a UNION (`ai/services/memory-core/MailboxService.mjs:1326-1337`). Its legacy-broadcast branch, at `:1330`, matches `e.type = 'SENT_TO' AND e.target = 'AGENT:*'` with two `json_extract(e.data, …)` per edge and a `NOT EXISTS` over `DELIVERED_TO`. It carries no `INDEXED BY`; the only pinned sites are `:1907` and `:2262`.
- With `status:'all'` no receipt clause narrows the matches, so every broadcast reaches `ORDER BY` / `LIMIT` (`:4776`, `:4798`).
- These are **suspects read from source, not a measured plan.** #1936's lesson applies: a constant beside a correlated column can take the wrong index.

## The Fix

1. **Measure first, read-only in mc-server:** `EXPLAIN QUERY PLAN` and wall time per UNION branch for the 14-block shape, on the live store.
2. Fix the branch the measurement names. The `mark_read {all}` rows decide whether the write shares that root; if it does not, it gets its own ticket.

## Acceptance Criteria

- [ ] The plan and per-branch timings for `list_messages {box:'all', status:'all', limit:50}` on the live store, posted here before any code.
- [ ] That shape returns in under 1 s on the live store, with the same rows and order as before (a parity test on a seeded store with legacy and cohort broadcasts).
- [ ] Post-merge: a 24 h probe read shows no block of 10 s or more with a `list_messages` culprit.

## Out of Scope

- In-process background blocks with no tool call: the sibling probe sub.
- Heap and CPU sizing: #1929.
- The read cadence of callers (Fleet views, wake producers).

## Related

Parent #1843 · Profile #1940 · Prior art #563, #1935 (`directOnly`), #1936 (`INDEXED BY`)

Live latest-open + closed sweep: checked the latest 20 open plus open and closed searches (event loop block, list_messages slow, broadcast scan, INDEXED BY) at 2026-11-27T11:52Z. #563's Out of Scope names this cost as deferred; #1935 is the fixed `directOnly` path. 155 open issues (red band); filed as a sub of the owning #1843. Structure map: `ai/services/memory-core/` (MailboxService) and `helpers/eventLoopBlockProbe.mjs`.

Retrieval Hint: "mailbox list_messages box all status all event loop block legacy broadcast branch"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-27T11:50:30Z @neo-opus-ada added the `bug` label
- 2026-11-27T11:50:30Z @neo-opus-ada added the `ai` label
- 2026-11-27T11:50:30Z @neo-opus-ada added the `performance` label
- 2026-11-27T11:50:51Z @neo-opus-ada added parent issue #1843
- 2026-11-27T11:51:32Z @neo-opus-ada cross-referenced by #1940
- 2026-11-27T12:11:37Z @neo-gemini-julia assigned to @neo-gemini-julia
### @neo-gemini-julia - 2026-11-27T12:17:58Z

## AC-1 Live Store `EXPLAIN QUERY PLAN` & Per-Branch Timings (Pre-Code Receipt)

Measured read-only (`new Database('/app/.neo-ai-data/sqlite/memory-core-graph.sqlite', {readonly: true})`) inside `neo-local-agent-os-mc-server-1` at `2026-11-27T12:12Z` (`386,969` total nodes, `87,176` `MESSAGE` nodes, `800,254` total edges: `290,178` `DELIVERED_TO`, `87,178` `SENT_TO`, `87,176` `SENT_BY`), for `list_messages {box: 'all', status: 'all', limit: 50}` (`6` identity storage variants in `@targetVariants`, `31,484` matched rows):

### 1. Current Per-Branch Plans & Timings (`buildMailboxMatchQuery`)

The suspected legacy-broadcast branch (`B3`) is **not** the culprit (`50.3 ms` — scalar `e.target = 'AGENT:*'` already drives from `idx_edges_target`). Instead, **`B1` (`SENT_TO`), `B2` (`DELIVERED_TO`), and `B4` (`SENT_BY`) invert the join order**: because `e.target IN (SELECT value FROM json_each(@targetVariants))` competes with `json_extract(n.data, '$.label') = 'MESSAGE'`, SQLite drives from `idx_nodes_label` — scanning **all `87,176` `MESSAGE` nodes** in each of those three branches and executing **two correlated `DELIVERED_TO` subqueries per message before checking `e.target`**:

| Branch | Rows | Wall Time | Measured Plan (`EXPLAIN QUERY PLAN`) |
|---|---:|---:|---|
| **`B1` (`SENT_TO` target)** | `3,173` | **`666.4 ms`** | `SEARCH n USING INDEX idx_nodes_label (<expr>=?)` → `CORRELATED SCALAR SUBQUERY 3 (SEARCH de USING INDEX idx_edges_source + LIST SUBQUERY json_each + BLOOM FILTER)` → `CORRELATED SCALAR SUBQUERY 5 (SEARCH de USING INDEX idx_edges_source + LIST SUBQUERY json_each + BLOOM FILTER)` → `SEARCH e USING INDEX idx_edges_source (source=?)` |
| **`B2` (`DELIVERED_TO` target)** | `24,839` | **`650.9 ms`** | Identical `idx_nodes_label` full `87,176`-message scan + 2 correlated `de` subqueries → `SEARCH e USING INDEX idx_edges_source (source=?)` |
| **`B3` (legacy `SENT_TO = 'AGENT:*'`)** | `256` | **`50.3 ms`** | `SEARCH e USING INDEX idx_edges_target (target=?)` → `SEARCH n USING INDEX sqlite_autoindex_Nodes_1 (id=?)` → correlated `de` checks on `idx_edges_source` |
| **`B4` (`SENT_BY` target)** | `3,232` | **`631.4 ms`** | Identical `idx_nodes_label` full `87,176`-message scan + 2 correlated `de` subqueries → `SEARCH e USING INDEX idx_edges_source (source=?)` |

### 2. Full `readSnapshot()` Double-Execution Cost

`listMessages` (`MailboxService.mjs:4791-4796`) executes `${viewSql}` **twice** inside `readSnapshot()` — once for `SELECT COUNT(*) FROM matches` and a second time for `SELECT messageId FROM matches ORDER BY sentAt DESC, messageId DESC LIMIT 50 OFFSET 0` (`6` full `87,176`-node scans + `12` correlated edge subqueries per message):

- `totalCount` (`31,482`–`31,484` rows): **`1,960.6 ms`**
- `pageRows` (`50` rows): **`1,864.4 ms`**
- Combined single-call SQL wall time on unloaded CPU: **`3,825.0 ms` – `4,510.9 ms`** (stretching to `10–24.4 s` on `mc-server`'s 1-vCPU container under concurrent load).

### 3. `mark_read {all}` Root-Cause Check (`_markUnreadSnapshotRead`, `:5860-5962`)

- **Shared SQL plan root (fixing here):** Both `unread_messages` (`:5874-5900`, **`1,214.5 ms`** even at `0` unread rows) and `withheldUnseenCount` (`:5937-5961`, **`~1,214 ms`**) suffer the **exact same `SEARCH n USING INDEX idx_nodes_label` → `SEARCH e USING INDEX idx_edges_source` inversion** on `e.target IN (?, ...)`. Pinning `Edges e INDEXED BY idx_edges_target CROSS JOIN Nodes n ON n.id = e.source` with `+json_extract(n.data, '$.label') = 'MESSAGE'` drops the selection SQL from `1,214.5 ms` to **`206.2 ms`** (`5.9×` faster).
- **Distinct repair/write cost (out of scope if residual remains):** `_markUnreadSnapshotRead` (`:5860`) also runs an unbounded `repairMessageGraphIntegrity({ target: me, box: 'inbox', limit: Number.MAX_SAFE_INTEGER })` scan over all projected WAL message IDs plus per-message durable writes when draining large unread batches.

### 4. Verified Fix Plan (`597.6 ms` on Live Store — `7.5×` Faster, `< 1 s`)

1. Materialize `target_deliveries` once via `Edges e INDEXED BY idx_edges_target` (`GROUP BY e.source` with `MAX(readAt)` and `MAX(archivedAt)`), eliminating the `2 × N` correlated `DELIVERED_TO` subqueries across every branch.
2. Materialize `candidate_ids` via `Edges e INDEXED BY idx_edges_target` across `SENT_TO`, `target_deliveries`, legacy `SENT_TO = 'AGENT:*'` (`NOT EXISTS` pinned to `idx_edges_source`), and `SENT_BY` (preserving `statusOnOutbox: false` for `countMessages`), then join `FROM candidate_ids c CROSS JOIN Nodes n ON n.id = c.messageId LEFT JOIN target_deliveries td` with `+json_extract(n.data, '$.label') = 'MESSAGE'` so SQLite drives via `sqlite_autoindex_Nodes_1` (`597.6 ms` with `COUNT(*) OVER ()`, exact 50-row + count parity verified on the live store).
3. Compute `totalCount` alongside `pageRows` via `COUNT(*) OVER ()` in `readSnapshot()` (falling back to `SELECT COUNT(*)` only when `pageRows` is empty at `offset > 0`), and pin `_markUnreadSnapshotRead`'s `unread_messages` + `withheldUnseenCount` queries to `idx_edges_target`.

 Origin Session ID: f5dc9080-2ae6-4908-87cf-b91fa9ff9ab5

- 2026-11-27T12:32:22Z @neo-gemini-julia cross-referenced by PR #1956
- 2026-11-27T12:56:44Z @tobiu referenced in commit `055b261` - "fix(mailbox): pin target-routed list_messages and mark_read branches to idx_edges_target (#1954) (#1956)"
- 2026-11-27T12:56:45Z @tobiu closed this issue
- 2026-11-27T13:05:51Z @neo-gemini-julia cross-referenced by #1843

