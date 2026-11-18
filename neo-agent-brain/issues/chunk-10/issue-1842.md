---
id: 1842
title: Open-work producer runs seat searches serially and blocks on mailbox
state: CLOSED
labels:
  - bug
  - ai
  - performance
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-18T15:38:04Z'
updatedAt: '2026-11-18T21:28:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1842'
author: neo-gemini-julia
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
closedAt: '2026-11-18T21:28:09Z'
---
# Open-work producer runs seat searches serially and blocks on mailbox

## Context

On `2026-11-18`, `@tobiu` reported multi-minute delays before newly approved/CI-green PRs appear in Fleet Manager's `awaitingMerge` ("PRs to merge") menu. Live code inspection plus `mc-server` telemetry (`65,092` tool calls in 24h; `who_is_online` `3.52 s` avg / `42.55 s` max; `list_messages` `853 ms` avg / `12.75 s` max) verified three Brain-side bottlenecks compounding on top of GitHub's GraphQL Search indexing lag and the cockpit's 60 s poll cadence.

## The Problem

1. **Sequential GitHub searches across all seats (~31–45 serial HTTPS round-trips/pulse):** `ai/services/fleet/openWorkProducer.mjs` (`run()`, lines 411–456: `for (const {seat, login, query, apiBase} of seats.filter(seat => seat.query))`) iterates **serially** across every seat, executing 3 sequential GitHub GraphQL `search` calls per seat (`authored`, `held`, `ended`), even though each seat authenticates with its own independent GitHub PAT and rate-limit bucket (#1535).
2. **`who_is_online` pre-step stalls every pulse behind `mc-server`:** `ai/services/fleet/wireFleetOpenWorkSource.mjs` (`readRows`, lines 323–328) awaits `readPresence()` (`who_is_online`, measured at `14.6 s–42.5 s` under load today) *before* starting per-seat GitHub reads in `producer.pulse()`.
3. **In-memory `awaitingMerge` blocked behind two `mc-server` mailbox scans:** `ai/services/fleet/FleetControlBridge.mjs` (`fleetOpenWork()`, lines 1468–1488) awaits `this.openWorkSource.readOpenWork()` and *then* awaits `Promise.all([readOwnQuestions(...open), readOwnQuestions(...expired)])`, both hitting `observeMessages({box: 'inbox', status: 'all', includeArchived: true})` on loaded `mc-server`. Whenever `mc-server` exceeds FM's `10 s` `livenessReadTimeout`, `fleetOpenWork` drops even though the in-memory `awaitingMerge` snapshot was already computed, surfacing as `unanswered` and leaving old/missing merge rows until the next poll.

## The Architectural Reality

- `ai/services/fleet/openWorkProducer.mjs` (`run`, lines 411–456): sequential `for (const ... of seats)` loop mutating shared `tally`, `seatReads`, and `reads`.
- `ai/services/fleet/wireFleetOpenWorkSource.mjs` (`wireFleetOpenWorkSource`, lines 313–362): `readRows` awaits `readPresence()` un-bounded inside `seatReaders` before GitHub queries dispatch.
- `ai/services/fleet/FleetControlBridge.mjs` (`readOwnQuestions` lines 132–188; `fleetOpenWork` lines 1468–1488): joins `openWork` and two `mc-server` `observeMessages` queries under one `await` with no axis timeout budget.
- `ai/configBase.mjs` (line 619): already defines `fleet.rosterAxisTimeoutMs` (`4000 ms`) for partial-degradation timeouts in `FleetManager#fleetPresenceStatus` / `FleetManager#fleetWakeStatus` (`{timeoutMs = AiConfig.fleet.rosterAxisTimeoutMs}`).

Design authority: `FleetControlBridge.mjs#fleetOpenWork` JSDoc ("The source envelope passes through untouched; an unwired source is named as unavailable, never as no open work. Beside it, questions counts the viewer's open questions...") and `FleetManager.mjs#fleetPresenceStatus` / `#fleetWakeStatus` ("4s leaves the roster's own work inside that budget, so a loaded plane costs the roster an axis, never the whole answer").

## The Fix

1. **Concurrent distinct-`query` seat reads (`ai/services/fleet/openWorkProducer.mjs`):** Run distinct-`query` seat lanes concurrently (`SEAT_CONCURRENCY` bounded worker pool; seats sharing an identical `query` function reference stay serialized on that lane) and replay per-seat `recordQuota`, `tally.pages`, `seatReads`, and `reads` deterministically in seat declaration order after settling. Also memoize `createQuery(token)` per token inside `seatReaders()` in `wireFleetOpenWorkSource.mjs`.
2. **Axis timeout on `readPresence()` (`ai/services/fleet/wireFleetOpenWorkSource.mjs`):** Default `presenceTimeoutMs = AiConfig.fleet.rosterAxisTimeoutMs` directly on `wireFleetOpenWorkSource` (per ADR-0019 `#17481` / B5 and `FleetManager.mjs:1078,1156`, avoiding entrypoint pass-through or C1 env re-derivation) and race `readPresence()` against it, updating `identityRows` if a late response settles while returning retained `{agents: identityRows}` on timeout so GitHub reads proceed unblocked.
3. **Axis timeout on `readOwnQuestions` (`ai/services/fleet/FleetControlBridge.mjs`):** Add an optional `{timeoutMs = null}` bound on `readOwnQuestions` and pass `{timeoutMs = AiConfig.fleet.rosterAxisTimeoutMs}` from `fleetOpenWork()` (keeping direct `fleetOwnQuestions` calls unbounded). On timeout, degrade only the timed-out question axis (`state: 'unavailable'`, timeout reason) while returning `openWork` (`awaitingMerge` and per-seat PR chips) immediately within FM's `10 s` read window.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `createOpenWorkProducer` / `pulse()` (`ai/services/fleet/openWorkProducer.mjs`) | #1535 per-seat PATs | Concurrent distinct-`query` seat lanes (`concurrency = SEAT_CONCURRENCY`), replaying quota tallies, `seatReads`, and `reads` in deterministic seat order | `concurrency = 1` or shared `query` reference stays serialized | JSDoc on `createOpenWorkProducer` | `openWorkProducer.spec.mjs` |
| `wireFleetOpenWorkSource` (`ai/services/fleet/wireFleetOpenWorkSource.mjs`) | #779 / ADR-0019 `#17481` / `rosterAxisTimeoutMs` | Bound `readPresence()` by `presenceTimeoutMs = AiConfig.fleet.rosterAxisTimeoutMs` | Return retained `{agents: identityRows}` when timed out | JSDoc on `wireFleetOpenWorkSource` | `wireFleetOpenWorkSource.spec.mjs` |
| `FleetControlBridge#fleetOpenWork` (`ai/services/fleet/FleetControlBridge.mjs`) | #922 / #1689 / ADR-0019 `#17481` / `rosterAxisTimeoutMs` | Bound `readOwnQuestions` sub-reads inside `fleetOpenWork` via `{timeoutMs = AiConfig.fleet.rosterAxisTimeoutMs}` | Return `openWork` (`awaitingMerge`) intact with `questions.state = 'unavailable'` | JSDoc on `readOwnQuestions` & `fleetOpenWork` | `wireOperatorComposeWriter.spec.mjs` |

Decision Record impact: aligned-with ADR-0019 (`#17481` non-entrypoint `import AiConfig` at the consumer use site + zero C1/B3/B5/HIDDEN-DEFAULT violations).

## Acceptance Criteria

- [ ] Unit tests in `test/playwright/unit/ai/services/fleet/openWorkProducer.spec.mjs` prove `createOpenWorkProducer.pulse()` executes distinct-`query` seats concurrently (barrier across all distinct seats before resolving, bounded by `concurrency`) while keeping `quota` buckets, `firstOrdinal`/`lastOrdinal`, `seatReads`, and `snapshot.rows` deterministic under out-of-order resolution.
- [ ] Unit test in `test/playwright/unit/ai/services/fleet/wireFleetOpenWorkSource.spec.mjs` proves a stalled `readPresence()` falls back to retained `identityRows` at `presenceTimeoutMs` without blocking `producer.pulse()`, while a late response still refreshes `identityRows`.
- [ ] Unit test in `test/playwright/unit/ai/services/fleet/wireOperatorComposeWriter.spec.mjs` proves `FleetControlBridge#fleetOpenWork()` returns `openWork` (`awaitingMerge`) intact when `observeMessages` hangs past `{timeoutMs}`, setting `questions.state === 'unavailable'` and `questions.expired.state === 'unavailable'`.
- [ ] `npm run ai:lint-config-template-ssot` and `ProbeMutationDispositionGuard.spec.mjs` pass cleanly with zero ADR-0019 violations.

## Out of Scope

- GitHub GraphQL Search index latency itself.
- Cockpit push/SSE invalidation vs 60 s `LivenessCadence` interval (separate `neomjs/neo-agent-institution` enhancement if needed).
- `mc-server` CPU/heap starvation & SQLite indexing (#1843).

## Avoided Traps

- **Sharing one PAT or running cross-seat searches under a single account quota:** explicitly prohibited by #1535.
- **Passing `AiConfig` values down from entrypoints (B5) or adding `?.` / `??` fallbacks on `AiConfig` reads (B3 / HIDDEN-DEFAULT):** banned by ADR-0019 §3.

## Related

#1535, #922, #1689, #1841, #1843, neomjs/neo-agent-institution#551, neomjs/neo-agent-institution#959

---

Live latest-open + closed sweep: checked latest 20 open of 154 (`red` band) + open/closed searches (`openWork`, `awaitingMerge`, `fleetOpenWork`, `readOwnQuestions`, #1535, #922, #1689) at 2026-11-18T15:36:25Z; Out of Scope / Avoided Traps read on #1535 and #1689; no open ticket owns `openWorkProducer` serialization or `fleetOpenWork` mailbox timeout isolation.
A2A in-flight sweep: latest 30 messages read across all read states (`2026-11-18T14:37Z–15:35Z`); no peer claim on open-work latency.
MC sweep: `"Fleet Manager PRs to merge delay get_pr_lane_activity github sync latency"`, 8 results (merge queue coordination history only, no open-work serialization/timeout prior art).
Own-assignment sweep: 1 open (`#1230`), non-overlapping.
Structure map (`node ai/scripts/diagnostics/structureMap.mjs --files --loc`, exit 0): owning folder `ai/services/fleet`; no new files required.

unowned-rationale: Claimed by `@neo-gemini-julia`.

Origin Session ID: 613064f8-74aa-4baa-8ee7-6a0b8540f31b
Retrieval Hint: "openWorkProducer sequential seats readPresence stall fleetOpenWork readOwnQuestions awaitingMerge delay"

## Timeline

- 2026-11-18T15:38:31Z @neo-gemini-julia cross-referenced by #1843
- 2026-11-18T15:38:36Z @neo-gemini-julia added the `bug` label
- 2026-11-18T15:38:37Z @neo-gemini-julia added the `ai` label
- 2026-11-18T15:38:37Z @neo-gemini-julia added the `performance` label
- 2026-11-18T15:38:37Z @neo-gemini-julia added the `agent-os` label
- 2026-11-18T17:51:53Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-18T17:56:46Z @neo-gemini-julia cross-referenced by PR #1845
- 2026-11-18T21:02:21Z @neo-gemini-julia referenced in commit `5067996` - "fix(fleet): parallelize per-seat open-work reads and bound plane axes (#1842)"
- 2026-11-18T21:02:22Z @neo-gemini-julia referenced in commit `834e39c` - "fix(fleet): share boundRosterRead across plane axes and pair monotonic quota (#1842)"
- 2026-11-18T21:28:09Z @tobiu referenced in commit `aa68a62` - "fix(fleet): parallelize per-seat open-work reads and bound plane axes (#1842) (#1845)

* fix(fleet): parallelize per-seat open-work reads and bound plane axes (#1842)

* fix(fleet): share boundRosterRead across plane axes and pair monotonic quota (#1842)"
- 2026-11-18T21:28:10Z @tobiu closed this issue

