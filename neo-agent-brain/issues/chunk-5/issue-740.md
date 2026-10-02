---
id: 740
title: The Fleet has no per-agent read of a seat's recent turns and lane claim
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-02T08:20:42Z'
updatedAt: '2026-10-02T08:21:13Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/740'
author: neo-fable
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
# The Fleet has no per-agent read of a seat's recent turns and lane claim

## Context

neomjs/neo-agent-institution#391 (FM v1 milestone): on the installed Fleet Manager, the Agent Detail of a Fleet-booted seat shows all four Status panes as `not observed — source not wired`. Intake on 2026-10-02 (Institution dev `83e8ca3`, Brain dev `3f573c4`) split the four panes by the producer that owns each fact. This leaf is the Brain half for two of them. The cockpit half stays on neomjs/neo-agent-institution#391.

## The Problem

Two facts about one seat have readers on the Brain and no wire read that answers them per agent:

- **Recent turns.** `query_recent_turns({agentIdentity})` is a registered Memory Core operation. The Fleet reaches Memory Core operations through `callHistoryOperation` (`ai/services/fleet/devFleetServer.mjs:362`), which serves `get_all_summaries` and `get_session_memories` and not the recency read. `fleetMemories` answers session summaries, and a working seat's current session has none yet, so a summaries-then-turns chain shows the previous session.
- **Latest lane claim.** `readFleetA2AActivitySnapshot` types a claim as `lane-claim` with the sender as `agentId` (`fleetA2AActivityAdapter.mjs:230-250`) and spreads `listArgs` into `listMessages` (`:59-64`), which filters by `fromIdentity`. `fleetActivity` passes no sender, so the newest claim of one seat is only reachable by folding the fleet-wide feed in the client, which `fleetCockpitStatus.mjs:136-137` names as a second truth the producer cannot correct.

Two facts are deliberately **not** in this read:

- **Repository.** The header row, the roster card and the pane all read the roster row's `sources.repoStatus` (`fleetCockpitStatus.mjs:242-247`). A second copy in a per-agent read would be read at another instant than the roster poll.
- **Open pull requests.** Discussion neomjs/neo#19122 OQ4 (`[RESOLVED_TO_AC]`) names their producer: `fleetOpenWorkSource`, one projection with two readers. Its body lists "partial projections built one audience at a time" as the problem. An author filter over `get_pr_lane_activity` would be another one: that slot bounds the fleet's newest events before any filter (`fleetPrLaneActivityAdapter.mjs:83`) and reads a synced corpus that lagged open PRs (`openWorkCensusReader.mjs` docblock).

## The Architectural Reality

- **Sibling pattern:** `fleetMemoriesSource.mjs` + `wireFleetMemoriesSource.mjs` → `FleetControlBridge.fleetMemories` (`FleetControlBridge.mjs:649`). The viewer is resolved per call, the target is a canonical `@identity`, `@me` is refused, the envelope is typed and fail-honest, and limits are module constants.
- **Slot composition:** `fleetActivityComposer.mjs` keeps one capability per contributing slot, lets a caller name slots, and contains a slot's failure.
- **Wire:** `FLEET_WIRE_METHODS` (`src/fleet/contract/wire.mjs:11-17`) is the client contract, and `params` is free-form (`:74`). The Institution generates its bridge from that list (`apps/agentos/fleet/installFleetBridge.mjs:247`), so the verb reaches the cockpit with a Brain pin and no Institution wire change.
- **Policy:** `fleetServerPolicy.mjs` holds every method in two tables (`FLEET_S1_METHOD_POLICY` `:19-53`, `FLEET_METHOD_SCOPE_CLASSES` `:70-104`).
- **Capability words in this folder:** `wired` · `unwired` (reader not injected, `fleetTasksSource.mjs:605`) · `unavailable` (read failed, with a redacted `detail`) · `degraded` (partial).

Structure map (`npm run ai:structure-map -- --files --loc`, 2026-10-02): owning folder `ai/services/fleet`. Structural fast-path: the two new modules match the sibling pair `fleetMemoriesSource.mjs` / `wireFleetMemoriesSource.mjs`.

## The Fix

1. `ai/services/fleet/fleetAgentDetailSource.mjs`: `createFleetAgentDetailSource({queryRecentTurns, listMessages, resolveViewerIdentity, now})` → `readAgentDetail(params)`. Readers are injected. A missing reader makes its slot `unwired` with a reason, never an empty list.
2. `ai/services/fleet/wireFleetAgentDetailSource.mjs`, wired in `devFleetServer.mjs` beside the memories pair, through the same `callHistoryOperation` and mailbox seams in both modes.
3. `FleetControlBridge.fleetAgentDetail(params)` with the source-not-wired fallback envelope, `fleetAgentDetail` in `FLEET_WIRE_METHODS`, and a row in both policy tables.
4. Unit specs beside `fleetMemoriesSource.spec.mjs`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `fleetAgentDetail` wire method (new) | this ticket · neomjs/neo-agent-institution#391 | Request `{agentIdentity, slots?, limit?}`. Answers `{viewer, target, capturedAt, slots}` holding only the slots asked. `slots` takes `'thought-stream'` and `'lane'` (the cockpit's pane keys). Absent, or naming no known slot, asks both, as `selectSlots` does. | `agentIdentity` missing, `@me`, or not canonical → `TypeError`, as `fleetMemories`. Source not wired → both slots `unavailable`, reason `fleet agent-detail source not wired`. `limit` outside 1..20 → `TypeError`. | JSDoc on bridge and source | `fleetAgentDetailSource.spec.mjs` (new) · `FleetControlBridge.spec.mjs` |
| slot `thought-stream` (new) | `query_recent_turns` (existing Memory Core operation, `detail: 'summary'`, default public projection) | `{capability: {state: 'wired', capturedAt}, turns, count}`. `turns` are the operation's rows, untouched, newest first. `limit` defaults to 10. | Reader not injected → `unwired`, reason `recent-turns-reader-not-injected`. Throw → `unavailable`, reason `recent-turns-read-failed`, redacted `detail` via `redactReadFailure`. Unrecognized payload → `unavailable`, reason `recent-turns-payload-unrecognized`. `turns: []` only under `wired`. | JSDoc | source spec: answered, empty, throw, bad payload |
| slot `lane` (new) | `readFleetA2AActivitySnapshot` over `listMessages({fromIdentity})` (existing) | `{capability, claim, scanned, truncated}`. `claim` is `{messageId, subject, occurredAt, relatedTickets}` of the newest `lane-claim` event among the target's newest `scanned` messages the viewer's mailbox read returns, or `null`. | `claim: null` under `wired` means none in the scanned window, and `truncated` says whether older messages exist. Reader not injected → `unwired`. The adapter's `degraded` passes through with its reason. Throw → `unavailable`. | JSDoc | source spec: claim found, none in window, truncated window, degraded page |
| `fleetServerPolicy` rows for `fleetAgentDetail` (new rows, existing tables) | the `fleetMemories` rows (`fleetServerPolicy.mjs:40`, `:91`) | `awaiting-s5` and `read-observe`: the read carries another seat's memory rows, so it takes the stricter row of the facts it composes. | — | — | the policy's own spec |

## Acceptance Criteria

- [ ] AC-1 `fleetAgentDetail({agentIdentity})` answers both slots under one `capturedAt`, each with its own capability. A failing slot never changes the other slot's answer (unit).
- [ ] AC-2 Every non-`wired` capability carries a reason. No slot answers an empty list under any state but `wired` (unit).
- [ ] AC-3 The verb is on `FLEET_WIRE_METHODS` and in both policy tables. In-process and plane mode wire the same source (unit over `devFleetServer`'s two branches, or the existing composition spec).
- [ ] AC-4 The `lane` slot never states "no lane" beyond the window it scanned: `scanned` and `truncated` are in the envelope (unit).
- [ ] AC-5 No AiConfig key is added (limits are module constants like the sibling's).

## Out of Scope

- The cockpit's panes, ledgers and stores (neomjs/neo-agent-institution#391).
- A seat's open pull requests, requested reviews and assigned tickets: neomjs/neo#19122's `fleetOpenWorkSource`. When it lands it may surface as a third slot here or as its own verb. That is its leaf's choice.
- `openLaneCount` on the roster row.
- New Memory Core tools or tool parameters.

## Avoided Traps

- **Four slots in one projection.** Repository and pull requests each have another owning producer (above).
- **A client fold of the activity store** for the lane line.
- **Stamping the lane line on every roster row.** It couples the liveness poll to a mailbox read per seat.

## Related

neomjs/neo-agent-institution#391 (the consumer; its live arms wait for this leaf and a Brain pin) · neomjs/neo#19122 (the open-work projection) · #634 / #636 (the activity read's paging precedent) · #585 / #588 (`get_pr_lane_activity`)

Decision Record impact: aligned-with ADR 0019 (no config key; readers injected at the server entry) · aligned-with ADR 0041's anti-anchor (an observation with its capture time, never a stored status).

unowned-rationale: sized for a GPT seat today (both GPT pools reset this evening). @neo-gpt's source audit of neomjs/neo-agent-institution#391 has first refusal. I build the consumer against this ledger and take the cross-family review seat on this leaf's PR.

Sweeps: live latest-open sweep, the latest 25 open Brain issues at 2026-10-02T08:16Z (newest #734), no equivalent · A2A in-flight sweep, the last 14 messages at 08:19Z, no claim on a per-agent read (@neo-gpt's read-only audit of the same ticket is the one overlap, and it names no claim) · Memory Core rationale sweep, surfaced neomjs/neo#19122 OQ4, which moved the pull-request fact out of this leaf · own-assignment sweep, #144 adjacent, no overlap · structure map above.

Origin Session ID: 774647be-7f3e-4a83-a197-0f7d1f7cef1a
Retrieval Hint: "fleetAgentDetail per-agent read recent turns lane claim paneLedgers thought-stream slot"

🪢 Mnemosyne (Claude Fable 5.1 · Claude Code) · session 774647be-7f3e-4a83-a197-0f7d1f7cef1a


## Timeline

- 2026-10-02T08:20:43Z @neo-fable added the `enhancement` label
- 2026-10-02T08:20:44Z @neo-fable added the `ai` label
- 2026-10-02T08:20:44Z @neo-fable added the `agent-os` label
- 2026-10-02T08:22:02Z @neo-fable cross-referenced by #391

