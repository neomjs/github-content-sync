---
id: 438
title: 'A timed-out session summary is retried every sweep, with no backoff'
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-23T14:17:14Z'
updatedAt: '2026-09-24T11:43:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/438'
author: neo-opus-vega
commentsCount: 0
parentIssue: 64
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
# A timed-out session summary is retried every sweep, with no backoff

## Context

Local plane, 2026-09-23. The orchestrator image is `b99ea11`; `SessionService.mjs`, `capSessionsForSweep.mjs`, `sessionSummaryReceiptStore.mjs` and the memory-core `configBase.mjs` are identical at `dev` `ef13cdb`. One `summary` run held the heavy-maintenance lease from 12:38:25Z to 13:56:49Z (78 min), during which the starvation watchdog counted five waiters past 1 h: `tenant-repo-sync` (the corpus tenant's first ingest, #430), `dream`, `memory-summary-backfill`, `message-concept-harvest` and `graphlog-compaction`. The orchestrator log shows only `drift complete: candidates=5; processed=1`. The reasons are in the Memory Core file sink inside the orchestrator (`/app/.neo-ai-data/logs/mc-server-2026-09-23.log`):

| time (Z) | line |
|---|---|
| 13:32:38 | `Found 5 sessions to summarize. Processing in batches of 1...` |
| 13:38:39 | guardrail `timeout` for `f18d3aa0…` (95 memories, summary at 79), skipping summary |
| 13:44:40 | guardrail `timeout` for `688a2d10…` |
| 13:50:40 | guardrail `timeout` for `bf94c4a1…` |
| 13:56:40 | guardrail `timeout` for `522f6841…` |
| 13:56:49 | `1/5 done (a3b3061e…)`, a one-memory session, 9 s |

Afterwards `SummarizationJobs` (read-only query) holds all four as `failed`, and `f18d3aa0` carries `retry_count = 9`.

Observed: the timeouts, the timings and the row states. Inferred: that the same sessions time out again in the next sweep, from the code below plus `retry_count = 9`. The next sweep will show it.

## The Problem

A session whose synthesis fails becomes a candidate again on the very next sweep, and it fails the same way. Its memory count still differs from its summary's, the model and the prompt have not changed, and nothing remembers the failure except a counter no code reads. Each attempt costs up to twice `sessionSummaryTimeoutMs` (raw synthesis, then the degraded retry; about 6 min per session measured) while holding the exclusive heavy-maintenance lease. In this run four such sessions spent 24 of the 78 minutes producing nothing, and because candidates are sorted by recency and capped at five, they also took four of the five slots from sessions that would have summarized.

ADR 0022's count cap (`maxSessionsPerSummarySweep`, neomjs/neo#13592) bounds how many sessions one sweep drains, which assumes each attempt is short. A known-failing session breaks that assumption without breaking the cap.

## The Architectural Reality

- `findSessionsToSummarize` (`SessionService.mjs:358`) selects sessions whose summary is missing or whose memory count differs and sorts them by `lastActivity` descending (`:522`). `capSessionsForSweep` then keeps the first `maxSessionsPerSummarySweep` (5, `configBase.mjs:164`).
- `summarizeSession` runs raw synthesis and then a degraded retry under `sessionSummaryTimeoutMs` (180000, `configBase.mjs:186`). When both time out it logs the guardrail symptom and returns `null` (`:777`).
- `failSummarizationJob` (`:1472`) sets `status = 'failed'`. `claimSummarizationJob` (`:1380`) re-claims a `failed` row unconditionally (`:1434`) and increments `retry_count`. The only other `retry_count` site is an INSERT in `sessionSummaryReceiptStore.mjs:206`, so it is written and never read.
- A second way back in: `queueSummarizationJob`, fired by `Server.mjs:685` on every Streamable-HTTP disconnect, rewrites any row that is neither `completed` nor `in_progress` to `pending` with `expires_at = NULL`, and the pending drain claims it through the single-session path.
- The per-session lines go to the memory-core logger's file sink (`stderrMode: 'debug'`), not to the orchestrator's stdout, so the orchestrator log shows a silent 78-minute hold.

## The Fix

Give a failed job a not-before, and do not treat a session inside it as a candidate:

1. `failSummarizationJob` records a retry-after time that grows with consecutive failures up to a ceiling: 30 min, doubling, capped at 24 h. The two values are Memory Core config leaves beside `maxSessionsPerSummarySweep` and `sessionSummaryTimeoutMs` (ADR-0019): `summaryFailureBackoffBaseMs` (`NEO_MC_SUMMARY_FAILURE_BACKOFF_BASE_MS`) and `summaryFailureBackoffMaxMs` (`NEO_MC_SUMMARY_FAILURE_BACKOFF_MAX_MS`), read at the SQL binding. The doubling stops at the fewest steps that reach the ceiling, so the ceiling binds for any declared policy. `expires_at` is unused on `failed` rows today, so it can carry this without a schema change.
2. `summarizeSessions` drops candidates still inside their backoff **before** `capSessionsForSweep`, so the five slots go to sessions that can run.
3. `queueSummarizationJob` leaves a `failed` row inside its backoff untouched, so a reconnecting session cannot re-enter through the pending drain.
4. A successful summary clears the backoff.
5. The `drift complete` line, the one that reaches the orchestrator log, also reports `failed=` and `backoff=` counts.

## Decision Record impact

aligned-with ADR 0022 (heavy-maintenance scheduling fairness): this closes the gap its count cap leaves when attempts are known to fail.

## Acceptance Criteria

- [ ] **AC-1** A session whose synthesis failed is not re-attempted while inside its backoff: the drift sweep drops it before the cap, so the freed slot goes to the next drift candidate, and a disconnect does not re-queue it as `pending`. Unit witness with an injected clock.
- [ ] **AC-2** Consecutive failures grow the backoff up to the declared ceiling, the session is claimable again once the not-before passes, and a successful summary resets it. Unit witnesses for the default policy and for a non-default one resolved at construction (no mutation of the shared config), plus the leaves' defaults and env bindings.
- [ ] **AC-3** The `drift complete` line reports how many candidates failed in this sweep and how many were skipped for backoff. Unit witness on the line.
- [ ] **AC-4** *(deployed plane, `[L4-deferred — operator handoff needed]`)* After deployment, the four sessions above are not re-attempted inside their backoff, and no sweep's lease hold contains a guardrail timeout for a backed-off session. Residual-Owner: #64.

## Out of Scope

- **The 54 minutes between `Starting drift-detection` (12:38:27Z) and `Found 5` (13:32:38Z).** Neither measured candidate explains them. The full metadata scan is 21 + 2 pages at about 100 ms per page (41,034 memories, 3,971 summaries), and receipt recovery replayed nothing. The sweep emits no phase timings, so today's logs cannot attribute the gap. It is recorded as a defect-note and not claimed here.
- Why a 95-memory session times out at 180 s on the local model (timeout calibration, model capacity).
- A non-LLM fallback summary, which would end the loop by writing something: a design question with a quality cost.

## Avoided Traps

- **Raising `sessionSummaryTimeoutMs`.** Every futile attempt gets longer, and so does the lease hold.
- **Skipping only at claim time.** The backed-off session would still take one of the five capped slots, and the sweep would under-fill while fresh candidates wait.
- **Excluding failed sessions permanently.** A model or timeout change would strand them. The ceiling re-admits them.

## Related

#64 (parent: scheduling starvation) · #430 (the starved corpus ingest) · neomjs/neo#13592 (the count cap) · #224 / #239 / #415 (starvation receipts)

Live latest-open sweep: latest 20 open issues in `neomjs/neo-agent-brain` at 2026-09-23T14:15Z, none equivalent. Keyword sweeps across the org (`guardrail timeout`, `findSessionsToSummarize`, `summarizeSession`, `session summarization`, `summary sweep`) surface only closed predecessors: the count cap (neomjs/neo#13592), the lease monopoly (neomjs/neo#13586), the wedged child (neomjs/neo#15686) and bounded synthesis (neomjs/neo#12833, neomjs/neo#13904). None re-admits failed jobs with a backoff. A2A in-flight sweep (15 most recent, 14:16Z): no claim on the summary lane. MC sweep: `query_raw_memories` (6) and `query_summaries` (5) on the symptom returned the June 2026 timeout and 30-day-window work and the scheduler-rotation challenge, with no prior decision on retrying failed jobs. Own-assignment sweep: #434, #432, #430, #417, #64, #65, #23, none overlapping. Structure map: N/A, since the fix edits the existing `ai/services/memory-core/SessionService.mjs` and adds no file.

Origin Session ID: 603e5af2-9d35-4bfc-9852-038c4cf38568
Retrieval Hint: `query_raw_memories("session summary guardrail timeout retried every sweep, failed summarization job no backoff, heavy-maintenance lease held 78 minutes")`

Authored by Vega (Claude Opus 5.5, Claude Code) 🌿



## Timeline

- 2026-09-23T14:17:15Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-23T14:17:15Z @neo-opus-vega added the `bug` label
- 2026-09-23T14:17:15Z @neo-opus-vega added the `ai` label
- 2026-09-23T14:17:15Z @neo-opus-vega added the `agent-os` label
- 2026-09-23T14:17:29Z @neo-opus-vega added parent issue #64
- 2026-09-23T14:26:12Z @neo-opus-vega cross-referenced by PR #439
- 2026-09-23T14:39:40Z @neo-opus-vega cross-referenced by #440
- 2026-09-23T14:46:24Z @neo-opus-vega cross-referenced by #442
- 2026-09-23T15:11:26Z @neo-opus-vega cross-referenced by #444
- 2026-09-23T15:14:47Z @neo-opus-vega cross-referenced by PR #445
- 2026-09-24T11:40:53Z @neo-opus-vega referenced in commit `7e4fa6b` - "fix(memory-core): the failed-summary backoff policy is declared in the Memory Core config (#438)

Resolves review R1 on #439. The base delay and the ceiling were service-local
literals; they are now leaves beside maxSessionsPerSummarySweep and
sessionSummaryTimeoutMs (NEO_MC_SUMMARY_FAILURE_BACKOFF_BASE_MS / _MAX_MS),
read at failSummarizationJob's SQL binding. The doubling's clamp was a fixed
6, which reaches the cap only because 30 min x 2^6 exceeds 24 h; it is now
the fewest steps that reach the declared cap, so the cap binds for any policy.

Witness: a child process resolves a 1 min / 24 h policy at construction on its
own in-memory graph (no singleton mutation) and must read 1 min, 8 min, 24 h;
red with the fixed clamp (64 min) and with the literals (30 min). The leaf
defaults and both env bindings are asserted on a fresh isolated provider."

