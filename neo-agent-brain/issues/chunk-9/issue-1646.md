---
id: 1646
title: Seats report their plan capacity to the fleet
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-12T14:23:16Z'
updatedAt: '2026-11-12T18:40:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1646'
author: neo-opus-vega
commentsCount: 3
parentIssue: 1508
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
# Seats report their plan capacity to the fleet

## Context

Peers plan better when they know which seats draw on one subscription pool, how much of each pool's week is left, and when each pool resets. For Claude seats the weekly all-models window and the weekly per-model window count apart: a capped model (Fable) may use only a share of the total. The operator raised it on 2026-11-12, after open-work wakes reached seats that were out of quota: `approve-runs` for neomjs/neo#20819 and `triage-outside` for neomjs/neo#20818. The operator does not want a seat benched for a rate-limit wait. A capacity reading also gives the open-work rotation what it needs to skip such a seat until its reset, with nothing to undo (#1653).

## The Problem

The fleet already has the readers, but nothing feeds them:

- the throttle axis of the roster DTO, `ai/services/fleet/fleetThrottleStateAdapter.mjs` (neomjs/neo#15272): its JSDoc reads "no trustworthy throttle truth source exists in the platform yet", and the injected `resolveThrottleState` reader is never supplied;
- the FM telltales (neomjs/neo#15273; the Institution's `apps/agentos/util/Telltale.mjs`), which show `unknown` for every seat;
- the throttle axis that `who_is_online` reserves, which reads `source: null`.

Measured at `cfb998af`: `FleetManager#throttleStateOptions` is `null` (`ai/services/fleet/FleetManager.mjs:346`, spread at `:864`), and nothing in `ai/` or `src/` assigns it. The same `git grep` finds the definition, so it can find an assignment.

The adapter rejected harness files as "heterogeneous and undocumented". Since then the Claude Desktop harness gives each seat a documented usage reading through its own session-management tool: the 5-hour window, the weekly all-models window and the weekly per-model window, each with percent used and `resetsAt`, readable by the seat without an extra credential. A Gemini CLI quota refusal carries its reset time. For other harnesses no source is known, so they read `unobserved`.

Two designs stalled without a ticket: Mnemosyne's of 2026-10-05 (pool topology is the operator's deployment data held on the plane; each seat reports its own gauge at turn boundaries; a seat without a source reads unobserved; one read serves peers and the roster) and Vega's sweep of 2026-10-15 (the readers are built, the producer is missing).

## The Architectural Reality

- **The seam:** `fleetThrottleStateAdapter.mjs` takes `resolveThrottleState(agent)` and normalizes the answer to `none | overage | rate-limited | unknown` under a capability envelope (`wired/observed`, `degraded/partial`, `degraded/none`). `FleetManager#throttleStateOptions` injects it.
- **Presence:** each turn already writes presence (`record_turn_presence`; `add_memory` reports `presenceTerminal`), and `who_is_online` projects it.
- **Rotation:** `ai/services/fleet/devFleetServer.mjs:749` passes `readPresence().undeliverable` as `readUndeliverable` to `wireFleetOpenWorkWakes`, and `openWorkWakes.mjs#pickRotationSeat` skips a seat whose route reads `unreachable`. The skip itself is #1653.
- **Participation** (`ai/services/memory-core/recordParticipation.mjs`, `ai/scripts/fleet/participation.mjs`) is the team decision (`active` or `operator_benched`). Capacity is not participation and must not write it.
- **Design authority:** n/a. This adds the producer that the seam neomjs/neo#15272 shipped empty; no existing behavior is called wrong.

## The Fix

1. **The seat reports.** `record_turn_presence` takes an optional `capacity` reading: `{observedAt, source, model?, windows: [{kind: 'five-hour'|'weekly'|'weekly-model', model?, percentUsed, resetsAt?}], poolKey?}`. The Claude hook entrypoint fills it every turn from the Desktop profile's usage sample; a seat may also send its usage tool's windows, with `resetsAt`, through the same field. `poolKey` is an opaque hash the seat computes from its account key before the reading leaves the seat. A seat without a source sends nothing.
2. **Pool membership is derived, never entered.** Seats whose readings carry one `poolKey` share a pool. A window's `resetsAt` comes from a seat's own report and carries over while it lies ahead. Non-Claude seats read `unobserved`. Nothing in this ticket asks the operator to write data.
3. **One read.** `who_is_online` returns each seat's capacity as `observed | stale | unobserved` with `observedAt`, the windows and its pool, and each pool's newest windows (two seats on one pool cross-check each other). From the same reading, `resolveThrottleState` answers `rate-limited` while a binding window is at 100 % before its `resetsAt` (the all-models windows, or a model window for the model the seat runs), `none` below that, and `unknown` when unobserved or stale. The rotation skip is #1653.

## Intake (2026-11-12, at `cfb998af`)

The per-turn presence write is already hook-driven: `ai/mcp/server/memory-core/helpers/TurnPresenceHookWriter.mjs` reads the harness's hook payload and calls `record_turn_presence` over MCP. The Fix's transports (a) and (b) therefore meet at one writer. What each source can carry, measured on a Claude Desktop seat:

| Source | Reaches the plane via | 5-hour | weekly all-models | weekly per-model | `resetsAt` | pool membership |
|---|---|---|---|---|---|---|
| the Desktop app's own usage history in the seat's profile directory (samples of the 5-hour and 7-day percent and the account's org) | the presence hook, with no agent in the loop | yes | yes | no | no | yes, by account |
| the Desktop app's session-management usage tool, called by the seat | a seat-side write at lane boundaries | yes | yes | yes | yes | no |

Pool membership for Claude seats is derivable on the host: `ai/scripts/lifecycle/hooks/claude/seatModelGuardHook.mjs#accountSharedWithAnotherSeat` already decides from each seat's Desktop profile whether another seat uses the same account.

Prescription checked: `ai/mcp/server/memory-core/helpers/TurnPresenceHookWriter.mjs` owns the concern (the per-turn beacon already crosses from the seat to the plane). The derivation stays in `ai/services/fleet`.

**Intake update (2026-11-12 16:5xZ, at `528105e7`; the drift probe fired, so the full workflow ran).**

- **Drift:** `openapi.yaml`, `openWorkWakes.mjs` and `wireFleetOpenWorkWakes.mjs` moved after filing (#1644). The build map (comment 6828573803) was measured after that merge, at the current `dev`. #807 (PR #1652) builds the `readUndeliverable` map from `who_is_online`'s verbose rows.
- **§0, the beneficiary:** the filed Fix had the operator write pool topology on the plane host, a new obligation that needs the operator's read before any build. Narrowed: the seat derives its `poolKey`, resets come from seat reports, and operator-held topology (non-Claude pools, reset cadence) leaves this ticket for a follow-up under #1508 that starts with that read.
- **Prescription checked:** `ai/services/memory-core/TurnPresenceService.mjs` — owns the concern: it is the per-seat write path with its own `turnPresence` leaves, and a turn node closes every turn, so the reading gets its own per-seat node. `ai/scripts/lifecycle/hooks/claude/turnPresenceHook.mjs` — owns the source read: the entrypoint resolves everything and the writer resolves nothing. `ai/services/memory-core/WakeSubscriptionService.mjs` owns the projection, and `FleetManager#throttleStateOptions` the seam.
- **Core idioms (9.6):** `TurnPresenceService` is a `Neo.core.Base` singleton (`src/core/Base.mjs`, `src/Neo.mjs#setupClass`); the reading is one graph node written through `GraphService.upsertNode`, and no instance is created. The staleness bound is an AiConfig leaf read at the use site (ADR 0019 §5.1).
- **Split (16:59Z):** the rotation skip (the former AC-4) moved to #1653, blocked by #807: putting an exhausted seat into the undeliverable map would also defer reviewer, author and merge wakes, which `routeOf` gates, so the skip needs its own reader.
- **Verdict:** valid as narrowed here.

## Contract Ledger

| Target surface | Source of authority | Proposed behaviour | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `record_turn_presence`, extended with `capacity` | this ticket | carries an optional reading with `observedAt` | no reading: the seat reads `unobserved` | the tool's description and handbook | unit arms; a real seat's reading on the plane |
| `poolKey` on the reading | this ticket | an opaque per-account hash; seats sharing one share a pool | absent: the seat has no pool | tool description | unit arms with opaque keys |
| `who_is_online`, existing (throttle axis reserved) | the presence read's contract | per seat: capacity state, windows, `resetsAt`, pool; per pool: newest windows | stale past its bound | tool description | unit arms |
| `FleetManager#throttleStateOptions.resolveThrottleState`, existing seam (`FleetManager.mjs:346`) | neomjs/neo#15272 | answers from the reading | `unknown` when unobserved or stale | adapter JSDoc | adapter arms unchanged and green |

Decision Record impact: none; aligned with the S2 telltale taxonomy (neomjs/neo D#15249), whose throttle axis this feeds.

## Acceptance Criteria

- [ ] A Claude Desktop seat's reading reaches the plane through `record_turn_presence`, and `who_is_online` returns it with `observedAt`; a seat's own report adds the per-model window and `resetsAt`. A seat without a source reads `unobserved`, and a reading past its bound reads `stale`.
- [ ] No operator-written topology, and no account identifier, membership or billing fact in source, fixtures, tests, the plane's graph or a public artifact: the seat hashes its account key before sending, and fixtures use opaque pool keys.
- [ ] `resolveThrottleState` answers `rate-limited`, `none` or `unknown` as The Fix states, including a seat on a capped model with only its model window exhausted. The adapter's arms and the FM telltales consume it unchanged.
- [ ] Post-merge: after a plane cut, a receipt from real seats: their capacity states in `who_is_online` and their throttle verdicts on the Fleet roster.

## Out of Scope

- The rotation skip for an exhausted seat: #1653.
- Bench and participation (#28).
- Re-seating owned work whose holder is out of quota: a sibling leaf once this reading exists.
- Provider usage APIs polled with seat credentials.
- Operator-written pool topology and its cockpit form: a follow-up under #1508 that starts with the operator's read.
- API-key seats.

## Avoided Traps

- **Presence as availability.** An idle Claude Desktop seat looks offline but still wakes. Only an exhausted window takes a seat out of the rotation.
- **Benching for a rate limit.** The operator's input (2026-11-12): a bench is a team decision, and a quota wait ends by itself.
- **A percent without its reset.** It cannot be planned with, and it cannot end a skip.
- **Membership in source.** Who shares a pool is per plane and private.

## Related

#1508 (Fleet Manager MVP, which holds the seam's neomjs/neo#15272 as a sub; the parent link is the epic owner's call), #1653, #807, neomjs/neo#15272, neomjs/neo#15273, #28, #31, #1642, #1630, #1373.

Live latest-open + closed sweep: latest 20 of 156 open issues (red band) and closed searches (`throttle`, `capacity`, `quota`, `usage` in titles) at 2026-11-12T14:2xZ; the owning epic #1508 holds the seam's ticket, so this is filed as its producer leaf, and no equivalent or exclusion was found. A2A in-flight claim sweep: the latest 30 messages, all read states; no claim on this scope (#1614, closed not planned, concerned run approval). MC sweep: the 2026-10-05 and 2026-10-15 designs above, never filed. Own-assignment sweep: 15 open; #28 covers the bench only. Structure map: the owning folders are `ai/services/fleet` and `ai/services/memory-core`.

Not a `good first issue`: it needs the private plane and a seat-side harness source.

Origin Session ID: 6e9ca37f-2eb0-421a-8661-cd92e806cf2b

Retrieval Hint: "seat plan capacity weekly window resetsAt pool topology throttle seam resolveThrottleState rotation skip rate-limited"

— Vega (Opus 5.5, Claude Code) 🌿

## Timeline

- 2026-11-12T14:23:17Z @neo-opus-vega added the `enhancement` label
- 2026-11-12T14:23:18Z @neo-opus-vega added the `ai` label
- 2026-11-12T14:23:18Z @neo-opus-vega added the `architecture` label
- 2026-11-12T14:23:18Z @neo-opus-vega added the `agent-os` label
- 2026-11-12T14:23:27Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-12T14:31:19Z @neo-opus-ada added parent issue #1508
### @neo-opus-vega - 2026-11-12T14:33:16Z

**Handover, 2026-11-12 (owner Vega; my session `6e9ca37f-2eb0-421a-8661-cd92e806cf2b` ends here and my next one continues).**

- **State:** filed, self-assigned, intake recorded in the body ("Intake": the presence hook carries the Desktop profile's usage sample; Claude pool membership is derivable on the host through `seatModelGuardHook.mjs#accountSharedWithAnotherSeat`; the per-model window only from a seat-side report). Ada was asked for the parent link under #1508. Mnemosyne answered "keep it, it is yours" and reads the body after Friday's reset.
- **Next:** branch from `dev`, start in `TurnPresenceHookWriter.mjs` (the sample read and the optional `capacity` argument), then `who_is_online`'s capacity projection, the `resolveThrottleState` injection in `FleetManager` and the `readUndeliverable` skip.
- **Not done:** no code, and no topology CLI yet.

— Vega (Opus 5.5, Claude Code) 🌿

### @neo-opus-vega - 2026-11-12T15:56:00Z

**Build map, 2026-11-12 at `528105e7` (read-only).** The next session builds from this; nothing is built yet. Branch `agent/1646-seat-capacity` exists, empty.

**0. Prerequisite found while mapping, verified live.** The rotation's `readUndeliverable` is inert. `devFleetServer.mjs:749` reads `.undeliverable` from `readPresence()`, which asks `who_is_online({verbose: true})` (`planeWhoIsOnlineReader.mjs:27`, `hostWhoIsOnlineReader.mjs:27`). Only the terse answer carries `undeliverable` (`WakeSubscriptionService.mjs:915-920`). So today no unreachable seat is skipped: a seat whose verbose row reads `wake.state: 'undeliverable'` still gets picked. Fix shape: build the map from verbose rows. `openWorkBoot.spec.mjs:7-35` slices this part of devFleetServer with a fixed seam list, so a new free identifier needs a seam there. Defect-note sent; promote it or fold it in.

**1. The source.** Each Desktop seat has `<userData>/plan-usage-history.json`, `userData = resolveUserDataDir(CLAUDE_CODE_EXECPATH)` (`hooks/claude/seatModelGuardHook.mjs:75`).
- Shape: `{version: 2, samples: [{t: epoch ms, org: uuid, u: {fh: 5-hour %, sd: weekly %}}]}`, mode 0600. No reset times. Nothing in `ai/` reads it yet.
- Samples are sparse: a median gap of about 4 h, a maximum of about 53 h. Staleness decides a lot.
- `org` equals the session folder's scope, so seats sharing an `org` share a pool. Never read more than `lastKnownAccountUuid` from the profile's `config.json`, which holds tokens.

**2. The transport.**
- `TurnPresenceHookWriter.mjs:108` is config-free and reads only `NEO_AGENT_IDENTITY`.
- `recordTurnPresenceOverMcp.mjs:147-152` builds the arguments, so a new field goes there.
- The Claude hook `hooks/claude/turnPresenceHook.mjs:81-103` is where a seat-side read can live.
- The request schema is non-strict and strips undeclared args (`openApiValidator.mjs:119-121`): declare the field in `TurnPresenceRecordRequest` (`openapi.yaml:5158`).
- Turn nodes (`AGENT_TURN_PRESENCE:<id>:<turnId>`) end at every `add_memory` terminal, and `getFreshTurnPresence` exposes only six fields. The newest reading therefore needs its own read or node, not the turn node.

**3. The projection.**
- `who_is_online`: `WakeSubscriptionService.mjs:819-923`, rows in `_projectAgentLiveness` `:1179`. The throttle axis is reserved in `COMPOSED_HOST_AXES` `:74`, envelope `:991-1002`, per-row 'unknown' `:1065`.
- Specs that pin the unobserved state and must move: `WakeSubscriptionService.spec.mjs:3746-3771`, `:3795-3823`.

**4. The throttle seam.**
- `FleetManager.throttleStateOptions` `:346` is read by `fleetThrottleStatus()` `:861` with no timeout. Set it in `devFleetServer.mjs` right after `presenceStateOptions` `:466`.
- The adapter (`fleetThrottleStateAdapter.mjs:55`) calls `resolveThrottleState(publicRow)` sequentially and accepts only `none | overage | rate-limited | unknown`.
- Join key: `presenceIdentityForAgent` = `@${githubUsername ?? id}`.
- Memoize the presence read per snapshot: `planeWhoIsOnlineReader` single-flights only overlapping calls.
- Production `fleetServer.mjs` wires none of presence, throttle or open-work (only `managedRoot`, `:1281`).

**5. The rotation.**
- `pickRotationSeat` (`openWorkWakes.mjs:34`) skips `routeOf(seat) === 'unreachable'`. `routeOf` also gates reviewer and author wakes (`:179`) and pending merges (`:234`), so a rate-limit entry defers those too.
- The map is cached for 5 min (`wireFleetOpenWorkWakes.mjs:14`, `:249-258`).

**6. Pool topology.** Copy the `participation.mjs` + `recordParticipation.mjs` pattern: properties on the plane's AgentIdentity node, which `who_is_online` can read. A file registry like `seat-operators.json` is host-only and invisible to `who_is_online`.

— Vega (Claude Opus 5.5, Claude Code) 🌿

- 2026-11-12T16:21:51Z @neo-opus-grace cross-referenced by #1651
- 2026-11-12T16:25:51Z @neo-opus-grace cross-referenced by PR #1652
- 2026-11-12T16:54:10Z @neo-opus-vega referenced in commit `c4d5f54` - "feat(memory-core): who_is_online projects seat capacity and plan pools (#1646)

Verbose rows carry each seat's reading from TurnPresenceService#getCapacity,
and a pools map groups seats by their opaque poolKey: the members, plus per
window the newest observation any member reported, keeping a member's reset
that still lies ahead. The terse answer does not read capacity. The response
schema declares both."
- 2026-11-12T16:57:42Z @neo-opus-vega referenced in commit `cbdf97f` - "feat(fleet): the throttle axis reads each seat's plan capacity (#1646)

FleetManager#throttleStateOptions gets its first reader. capacityThrottleState
maps a who_is_online row's capacity onto the throttle taxonomy: rate-limited
while a binding window sits at 100 % (the 5-hour or weekly all-models window,
or a capped model's window when the seat runs that model), none below it, and
unknown when the reading is unobserved or stale or no binding window is still
current. The reader keeps one presence answer per snapshot, so the adapter's
per-agent calls cost a single read, and joins on the presence identity."
- 2026-11-12T17:00:14Z @neo-opus-vega cross-referenced by #1653
- 2026-11-12T17:02:51Z @neo-opus-vega cross-referenced by PR #1654
### @neo-opus-vega - 2026-11-12T17:07:14Z

**Handover, 2026-11-12 18:4xZ (owner Vega; session `9a092228-2906-430a-b145-0382cf7a48f3` ends; this replaces the 17:1xZ text).**

- **PR #1654** at `bb8830a8`, every check green. Euclid's Round 1 (review 5879207719, two P2 actions) is addressed (response 6831878168). Two different pool keys are two accounts: a newer report from another pool replaces every window, an older one is dropped, and the node's model, pool and source are the newest observation's. `withPoolWindows()` lays the pool's newest windows over a fresh member's own reading, so the throttle verdict reads the plan. Euclid was re-requested at 18:20Z; his Round 2 is a disposition.
- **AC-4** (the rotation skips a `rate-limited` seat until its reset) lives in #1653, unblocked since #1652 merged. Build it with its own reader, so owned work keeps routing through `routeOf`. The installed-plane receipt for this ticket rides on #1653.
- **Pickup:** answer Round 2 in place. On approval, the reviewer posts the merge handoff.

— Vega (Claude Opus 5.5, Claude Code) 🌿

- 2026-11-12T18:08:07Z @neo-opus-vega referenced in commit `bb8830a` - "fix(fleet): a capacity reading stays with its account, and members read their pool (#1646)

Round 1: an older report replaced the newer pool and model; a newer report
from another pool kept the old pool's windows and resets; and the throttle
reader ignored the pool's newest windows. Two different pool keys are now
two accounts: a newer report from another pool replaces every window, an
older one is dropped, and the node's metadata is the newest observation's.
The reader lays the pool's windows over a fresh member's own."

