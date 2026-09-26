---
id: 552
title: who_is_online reads and parses every MESSAGE node on each call
state: CLOSED
labels:
  - bug
  - ai
  - performance
assignees:
  - neo-opus-vega
createdAt: '2026-09-26T18:59:45Z'
updatedAt: '2026-09-26T19:48:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/552'
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
closedAt: '2026-09-26T19:48:54Z'
---
# who_is_online reads and parses every MESSAGE node on each call

## Context

Measured on the local plane on 2026-09-26 (mc-server at Brain dev `60f911e`), while every peer reported Memory Core calls timing out:

| Signal | Value |
|---|---|
| `who_is_online` duration (`get_memory_core_tool_metrics`, last hour) | min **6,445 ms**, avg 7,410 ms, 240 calls/h |
| mc-server main process CPU (`/proc/1/stat`, 10 s sample) | 901 ticks / 10 s ≈ 90 % of one core, sustained since the 10:49Z boot |
| queueing behind it | `list_messages` avg 5.1 s (max 34 s), `query_recent_turns` avg 11.2 s, `healthcheck` avg 9.9 s (max 188 s), `get_rem_pipeline_state` avg 8.0 s |
| health-axis timeouts (`REM axis … timed out after 1500ms`) | ~640/h from 09:03Z today; 3 in total on 09-25 |
| caller | the Fleet Manager cockpit's liveness tick: `who_is_online {verbose: true}` every 15 s from 09:03Z (240/h today, 2/h on 09-25); the cadence is Institution's side (linked below) |

At 18:58Z the MCP client reported the server unavailable. The 2 GiB container had not restarted.

## The Problem

`WakeSubscriptionService#whoIsOnline` derives the review-load trail by reading **every** MESSAGE node and parsing each row's full `data` blob in JS, on every call, with no cache:

```js
// ai/services/memory-core/WakeSubscriptionService.mjs:997
rows = sqlite.prepare(`
    SELECT data FROM Nodes
    WHERE id LIKE 'MESSAGE:%' AND json_extract(data, '$.label') = 'MESSAGE'
`).all();
for (const row of rows) { const properties = JSON.parse(row.data).properties ?? {}; messages.push({from, sentAt, subject, to}) }
```

The plane holds ~22.6k MESSAGE nodes (the `SENT_BY` count), so each call parses the whole mailbox to keep four fields; `deriveReviewLoad` (`helpers/reviewLoadProjection.mjs:57`) then applies `REVIEW_LOAD_TRAIL_HORIZON_MS` in JS, after the parse. better-sqlite3 is synchronous, so the 6.4 s floor blocks the event loop for every other request. A 15 s poller alone consumes ~43 % of the server; the presence roster itself is cheap.

## The Architectural Reality

- The trail is horizon-bounded by definition, and SQLite can apply the bound and the projection: `json_extract(data, '$.properties.sentAt') >= ?` plus four `json_extract` columns, no blob parse.
- The presence axis is a recency proxy sampled at turn boundaries (the tool's own `axes.presence.reason`); a short memo of the derived load changes nothing a caller can observe.
- `HealthService` already serves a cached status (`Using cached health status (age: 250s)` in the log); `who_is_online` has no equivalent.
- Owning folder per `npm run ai:structure-map -- --files --loc`: `ai/services/memory-core/` (`WakeSubscriptionService.mjs`) and `ai/services/memory-core/helpers/` (`reviewLoadProjection.mjs`); no new file.

## The Fix

1. Bound the trail read in SQL: select `from`, `sentAt`, `subject`, `to` via `json_extract` where `sentAt >= now − horizon`; drop the JS parse of the whole blob.
2. Memoize the derived review load for a short TTL (≤ 30 s) keyed on the bounded row count and max `sentAt`, so a poller pays the read once per window.
3. Keep `deriveReviewLoad` pure; its horizon parameter becomes the SQL bound's single source.

## Acceptance Criteria

- [ ] AC-1: on the local plane (~22.6k MESSAGE nodes) `who_is_online` min/avg duration in `get_memory_core_tool_metrics` drops below 300 ms under the cockpit's 15 s tick (before: 6,445 ms min).
- [ ] AC-2: a unit arm feeds messages inside and outside the horizon and asserts the `reviewLoad` map is identical to the pre-change derivation (red-first against a full-scan control).
- [ ] AC-3: mc-server main-process CPU under the same tick stays below 30 % (10 s `/proc/1/stat` sample), and no `REM axis … timed out` line is caused by the tick during a 10 min window.

## Out of Scope

- The cockpit's cadence and `verbose` use on the tick (Institution ticket, linked below).
- `get_rem_pipeline_state`'s own 8 s cost and the health axes' 1.5 s budget.
- The presence semantics of `who_is_online` (#31).

## Related

#31 · the Institution cockpit ticket filed beside this one · the health-axis timeouts in `HealthService`

Live latest-open sweep: checked the latest 20 open Brain issues at 18:57Z; no equivalent (#550/#547/#503/#513/#514 are wake-route items). A2A claim sweep: `list_messages` returned "server unavailable" at 18:58Z, the symptom this ticket describes; no `[lane-claim]` on this scope was seen in the 18:2xZ inbox read. Memory sweep: `query_raw_memories` degraded (chroma socket hang up), no result. Own-assignment sweep: none of my open Brain tickets covers it.

Origin Session ID: 27467eea-851e-486b-a0ca-55f744b67fdf
Retrieval Hint: "who_is_online MESSAGE scan review-load trail horizon SQL bound mc-server CPU cockpit poll"

## Timeline

- 2026-09-26T18:59:45Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-26T18:59:46Z @neo-opus-vega added the `bug` label
- 2026-09-26T18:59:46Z @neo-opus-vega added the `ai` label
- 2026-09-26T18:59:46Z @neo-opus-vega added the `performance` label
- 2026-09-26T19:10:04Z @neo-opus-vega cross-referenced by #64
- 2026-09-26T19:12:29Z @neo-opus-vega cross-referenced by PR #553
- 2026-09-26T19:24:03Z @neo-opus-vega referenced in commit `fe42305` - "fix(memory-core): who_is_online reads the mailbox trail through a bounded, indexed projection (#552)

The review-load trail read selected every MESSAGE node and parsed each blob in JS on every
call; the projection then dropped everything older than its horizon. The read now bounds
itself to that horizon in SQL and projects the four fields it reads with one json_extract
per row. The storage declares two idempotent expression indexes: a partial sentAt index
over MESSAGE rows for that read, and a label index for every json_extract($.label) read."
- 2026-09-26T19:36:09Z @neo-opus-vega cross-referenced by #554
- 2026-09-26T19:48:54Z @tobiu referenced in commit `b0b85e8` - "Merge pull request #553 from neomjs/vega/552-who-is-online-trail-read

fix(memory-core): who_is_online reads the mailbox trail through a bounded, indexed projection (#552)"
- 2026-09-26T19:48:54Z @tobiu closed this issue

