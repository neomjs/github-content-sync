---
id: 1622
title: The roster's retained presence read expires before the next 60 s poll
state: CLOSED
labels:
  - bug
  - ai
  - regression
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-12T01:16:35Z'
updatedAt: '2026-11-12T02:02:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1622'
author: neo-opus-ada
commentsCount: 1
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
closedAt: '2026-11-12T01:52:41Z'
---
# The roster's retained presence read expires before the next 60 s poll

## Context

Operator report, 2026-11-12 ~01:05Z, on the installed Fleet Manager: the header reads `0 started · 13 UNKNOWN` while the roster shows who is online, and **Hide benched** hides nothing. This is #1586's symptom again, after its fix (PR #1592) shipped.

What I observed on the installed FM. Its organism is Brain `7219f4e1`, Institution `6f00d3da` and engine `c7e19744`, and the plane's `deployedRevision` is `7219f4e1`, so both halves of #1592 are live.
- **Roster store:** a Neural Link read of `neo-state-provider-1__fleetRoster` at 01:09Z. All 13 rows carry `participationStatus: null` and `participationRead: {state: 'unread', reason: 'presence read still pending after 4000 ms'}`. Every row's `sources.runtime.state` is `wired`.
- **The Fleet's `main.log`:** all 270 presence reads it holds since 11-11 20:17Z answered after the 4,000 ms bound (`[fleet] roster presence read answered after … ms`). The minimum is 4,460 ms, the median 6,438 ms and the maximum 29,599 ms. The rotated log shows 45–58 late reads in most hours, back to 11-09 21:00Z.
- **Plane metrics, last hour:** `who_is_online` had 132 calls, avg 2,602 ms and max 11,411 ms. On each 60 s cockpit tick it starts within 2 ms of `query_recent_turns` (01:10:54.845Z / .847Z), which averages 5,147 ms (max 17,015 ms).

## The Problem

#1592 keeps a late presence answer for `4 * timeoutMs` (16 s), so that the next poll can serve it. Its JSDoc sizes that window for "11–15 s synchronized cockpit-poll stalls across up to three 5 s roster ticks". The cockpit has no 5 s roster tick. It reads the roster every 60 s, checked on a 15 s pass (`LivenessCadence.DEFAULT_INTERVALS.roster = 60000`, `DEFAULT_PASS = 15000`, set by `neomjs/neo-agent-institution#255` on 2026-09-26).

So each read times out at 4 s and finds the previous tick's answer 45–60 s old. It discards that answer and stamps participation `unread`. The retention never fires, and #1586's post-merge check (AC-4) fails on the deployed plane.

Two consumers break on the same `null`:
- **Hide benched** filters `participationStatus === 'operator_benched'` (Institution `apps/agentos/view/fleet/roster/Controller.mjs`, `seatViewOrdering`). None of the 13 rows match, and the three benched seats render as offline.
- **The start summary** (`FleetStartPlan`) counts a seat UNKNOWN when its participation is unread, and excludes only a seat with a known non-active status. So all 13 are UNKNOWN and none are excluded.

## The Architectural Reality

Design authority: `"a read still out at timeoutMs serves the active reader's latest settled observation while within 4 * timeoutMs (… spanning 11–15 s synchronized cockpit-poll stalls across up to three 5 s roster ticks while bounding staleness …)"` (the `FleetManager#fleetPresenceStatus` JSDoc). The intent is to carry the latest settled observation to the next roster read; the premise used to size it is false. The cadence authority is `"the roster and the activity stream within a minute"` (`AgentOS.util.LivenessCadence`).

- **The mechanism already exists.** `boundRosterRead(read, axis, timeoutMs, {maxRetainedAgeMs, now, readRetained, writeRetained})` in `ai/services/fleet/FleetManager.mjs` holds it; only the window is wrong.
- **The window is a policy literal:** how old a settled presence answer may be before the roster calls participation unread. ADR-0019 §5.2 puts such a literal in a declarative leaf read at its use site, beside `fleet.rosterAxisTimeoutMs` in `ai/configBase.mjs`.
- **Structure map:** the owning folder is `ai/services/fleet`; no file is added or moved.

## The Fix

1. Declare `fleet.rosterRetainedMaxAgeMs` in `ai/configBase.mjs` as `leaf(120000, 'NEO_FLEET_ROSTER_RETAINED_MAX_AGE_MS', 'number')`, and add it to `ai/scripts/lint/config-leaf-parity.json`. The leaf comment carries the derivation: at least one roster interval (60 s) + one pass (15 s) + the bound (4 s) = 79 s; 120 s leaves room for one late tick.
2. In `FleetManager#fleetPresenceStatus`, pass `maxRetainedAgeMs: AiConfig.fleet.rosterRetainedMaxAgeMs`, read at the use site, instead of `4 * timeoutMs`. Rewrite the JSDoc sizing sentence to the 60 s cadence.
3. Add two specs to `test/playwright/unit/ai/services/fleet/FleetManager.spec.mjs`:
   - (a) Poll 1 times out and settles at +6 s. Poll 2 at +60 s times out and serves poll 1's `operator_benched` participation, with `capability.capturedAt` taken from `payload.generatedAt`. Red on `dev`.
   - (b) A gap past the window returns `unread`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `AiConfig.fleet.rosterRetainedMaxAgeMs` (new leaf, `ai/configBase.mjs`) | ADR-0019 §5.2; `LivenessCadence` roster 60 s / pass 15 s | How long the roster serves a settled presence answer after a read times out; default 120 s | env `NEO_FLEET_ROSTER_RETAINED_MAX_AGE_MS` | the leaf comment | config-leaf parity lint |
| `FleetManager#fleetPresenceStatus` (`ai/services/fleet/FleetManager.mjs`) | #1586 / PR #1592 | A timed-out read serves the newest settled presence payload up to the leaf's age; past it, participation stays `unread` and presence `unknown` | unchanged when no settled payload exists | JSDoc | specs (a) and (b) |

## Decision Record impact

`aligned-with adr:0019-aiconfig-reactive-provider-ssot`

## Acceptance Criteria

- [ ] `fleet.rosterRetainedMaxAgeMs` is a declarative leaf (120,000 ms, `NEO_FLEET_ROSTER_RETAINED_MAX_AGE_MS`) read at its use site in `fleetPresenceStatus`. `4 * timeoutMs` and the "5 s roster ticks" premise are gone from code and JSDoc, and the config-leaf parity lint passes.
- [ ] Spec (a) is red on `dev` and green at the head; spec (b) passes on both.
- [ ] *(Post-merge; needs an FM build from a Brain checkout at the fix, plus its install)* The live roster store shows `operator_benched` on `neo-kimi-phoebe`, `neo-kimi-iris` and `neo-preview`. Hide benched hides those three cards. The start summary counts them as excluded and no longer reports every seat UNKNOWN.

## Out of Scope

- Making `who_is_online` or `query_recent_turns` faster, or spreading the cockpit's reads across its tick (Institution `LivenessCadence`). Both would shorten the late window; neither removes the need to carry an answer across a 60 s gap.
- Raising `fleet.rosterAxisTimeoutMs`. Per #1586's Avoided Traps, the bound protects the cockpit's 10 s first-read budget (`neomjs/neo-agent-institution#841`).
- The wake axis. Its reads answer past the bound too (5,886–12,238 ms in the same logs); whether its consumers degrade the same way is not measured here.

## Avoided Traps

- **Deriving the window from `timeoutMs` again.** The bound and the cadence are independent, and tying one to the other is how the window ended up at 16 s.
- **Reading the cockpit's cadence across the repo boundary.** Brain cannot import Institution. The leaf states Brain's own tolerance, documented against today's cadence.
- **Unbounded retention.** With the plane gone, the roster must still degrade to `unread` within the window.

## Related

- #1586 / PR #1592: the predecessor; its AC-4 fails live. #1468 / PR #1469: the 4 s bound.
- `neomjs/neo-agent-institution#255` (the 60 s roster cadence) and `neomjs/neo-agent-institution#841` (the 10 s first-read budget).

Live latest-open + closed sweep: latest 20 open (153 open, red band; `#1619`..`#1510`) plus a closed search (`presence read timeout participation Hide benched`; #1586 read in full, its Out of Scope and Avoided Traps kept) at 2026-11-12T01:14Z; no open equivalent.
A2A in-flight claim sweep: latest 30 messages across all read states at 01:14Z; no claim on this surface.
MC sweep: `Hide benched button does nothing; fleet header shows every seat UNKNOWN while the roster shows who is online`, 6 results (Julia's #1586 diagnosis, 11-10/11); no later decision.
Own-assignment sweep: 5 open in Brain (#1618, #1531, #571, #133, #1508), none on this surface.

Ownership: self-assigned, @neo-opus-ada.

Origin Session ID: a84af7d7-9095-4958-8288-503b774a5f6b
Retrieval Hint: "retained presence read expires before the next 60 s roster poll: Hide benched participation unread, 13 UNKNOWN start summary, rosterRetainedMaxAgeMs"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-12T01:16:35Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-12T01:16:37Z @neo-opus-ada added the `bug` label
- 2026-11-12T01:16:37Z @neo-opus-ada added the `ai` label
- 2026-11-12T01:16:37Z @neo-opus-ada added the `regression` label
- 2026-11-12T01:16:37Z @neo-opus-ada added the `agent-os` label
- 2026-11-12T01:16:48Z @neo-opus-ada cross-referenced by #1586
- 2026-11-12T01:22:04Z @neo-opus-ada cross-referenced by PR #1623
- 2026-11-12T01:52:41Z @tobiu referenced in commit `b991bd5` - "fix(fleet): a late presence answer lasts to the next roster read (#1622) (#1623)

The cockpit reads the roster once a minute, so #1592's 4 * timeoutMs
(16 s) window discarded every late presence answer before the next
read could serve it, and each seat's participation read unread: Hide
benched matched nothing and the start summary counted all seats
UNKNOWN. The window is now its own leaf, fleet.rosterRetainedMaxAgeMs
(120 s), read where the presence read is bounded."
- 2026-11-12T01:52:42Z @tobiu closed this issue
### @neo-opus-ada - 2026-11-12T02:02:29Z

**AC-3 met live, 2026-11-12 02:02Z.** The installed FM is now Brain `b991bd5` + Engine `be8d0b0` + Institution `499339f`. The install was rehearsed, matched Electron 43.8.1 and the CDHash, and sealed continuity in 5.9 s with all 8 running seats verified. Recovery brought them back `active`.

- **Roster store** (`neo-state-provider-1__fleetRoster`, Neural Link): all 13 rows have `participationRead: read`. `participationStatus` is `operator_benched` on `neo-kimi-iris`, `neo-kimi-phoebe` and `neo-preview`, and `active` on the other ten. That is what Hide benched filters on.
- **Start summary** (`fleet-start-summary`): "8 started · 3 excluded", where it read "0 started · 13 UNKNOWN".
- **Presence reads** still answer late (6.5–6.9 s against the 4 s bound). The 120 s window now carries those answers across the 60 s poll, as designed. Speeding up the read stays out of scope here.

This also clears #1586's AC-4.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-12T02:56:37Z @neo-opus-ada cross-referenced by #1627

