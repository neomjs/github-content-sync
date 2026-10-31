---
id: 1400
title: who_is_online never sees a peer's turn-presence beacon
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-31T17:01:48Z'
updatedAt: '2026-10-31T17:04:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1400'
author: neo-fable-mnemosyne
commentsCount: 0
parentIssue: 31
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
# who_is_online never sees a peer's turn-presence beacon

## Context

2026-10-31 ~16:50Z the operator, watching every seat in a turn, read my recovery line "Julia is idle" and said no peer was idle. `who_is_online({verbose: true})` at 16:51:17Z: ten active seats; **my row carries `signals.turnPresence`** (turn started 16:50:52Z); **the other nine carry `turnPresence: null`**; four read `idle` on add_memory recency alone (Julia's last write 15:50Z, Euclid 16:20Z, Sophie 16:26Z; Grace flipped to `online` at 16:50Z by a write). The plane's own log for the day at 16:57Z: **5,844 `record_turn_presence` calls** — 5,373 `claude-post-tool-use` progress, 261 `claude-user-prompt-submit` start, 210 `codex-user-prompt-submit` start. The writers deliver. The reader does not see them.

## The Problem

The beacon exists to rescue a mid-turn seat whose `add_memory` has aged past the 15-minute freshness window (`TurnPresenceService.getFreshTurnPresence` JSDoc, lines 188–194; `WakeSubscriptionService` lines 1211–1224: "rescues the verdict BEFORE any not-online conclusion"). That rescue fires **only for the caller's own row**:

1. the roster verdict runs `TurnPresenceService.getFreshTurnPresence(identity)` inside the **caller's** request context;
2. `_findNewestActiveTurnId` (lines 272–296) is a raw SQL over `Nodes` — unscoped, it finds the peer's newest active `turnId`;
3. `_getTurnPresenceProperties` (lines 261–263) then reads the node through `GraphService.getNodeRecord`, which applies `isRlsVisible` (GraphService lines 31–49): visible when the owner key is null, equals the requester, `sharedEntity` is true, or `visibility` is `team`;
4. a beacon is written with `userId: normalizeUserId(agentIdentity)`, `sharedEntity: false` and no `visibility` (lines 154–155) — so for every requester but its owner the read returns `null`, `getFreshTurnPresence` returns `null`, and the row falls to add_memory recency.

Result: every peer reads every other peer by turn-boundary recency alone — the inversion #31 names as problem 1 ("the busiest seat looks stalest"), now with the rescue built and blind. A Gemini review turn or a GPT generation turn of 20–40 minutes reads `idle` while the seat works; a seat on its first turn reads `neverConnected`. The single-identity case passes (owner == requester), which is why the surface looked wired.

The sibling signal, `activityRecency`, was made roster-scoped across identities in June (neo#13527 / neo#13557, `_readActivityRecency`); the beacon path (neo#13515, Substrate B) never got the same treatment.

## The Architectural Reality

- `ai/services/memory-core/TurnPresenceService.mjs` — `recordTurnPresence` (85–185: owner key + `sharedEntity: false`), `getFreshTurnPresence` (208–228), `_getTurnPresenceProperties` (261–263, the scoped read), `_findNewestActiveTurnId` (272–296, the unscoped SQL that already holds the row's `data`).
- `ai/services/memory-core/GraphService.mjs` — `isRlsVisible` (31–49), `getNodeRecord` (1234–1255, the RLS return-boundary check), `getUnscopedNodeRecord` (1281+, warns on every request-path call — not the fix).
- `ai/services/memory-core/WakeSubscriptionService.mjs` — the row verdict and the mid-turn rescue (1211–1224); its `signals.turnPresence` is what the verbose wire and the Fleet's `fleetPresenceStateAdapter` grade from (#318's vouched horizons).
- Structure map (`npm run ai:structure-map -- --files --loc`): owning folder `ai/services/memory-core`, siblings `TurnPresenceService.mjs` / `WakeSubscriptionService.mjs`; no new file.

## The Fix

In `TurnPresenceService`, let the one query carry the answer: `_findNewestActiveTurnId` already selects the row's `data` — return the parsed `properties` (rename to `_findNewestActiveTurn`) and project `getFreshTurnPresence` from them. One SQL read, no second cache-scoped lookup, the RLS boundary untouched. `recordTurnPresence` keeps `_getTurnPresenceProperties` for its own `...current` merge (owner == requester, visible). Not `getUnscopedNodeRecord`: its misuse tripwire warns inside every bound request, and the roster verdict is a request path.

Projection unchanged — `turnId`, `startedAt`, `lastProgressAt`, `fresh`, `freshUntil`, `expiresAt`; the beacon's `note` (tool names) stays server-side. Visibility class = the one `activityRecency` already publishes per roster row (last write time, cross-identity, roster-scoped); the roster read's tenant scoping is #138's and this leaf does not widen across tenants.

Spec (`test/playwright/unit/ai/services/memory-core/TurnPresenceService.spec.mjs` or the WakeSubscriptionService spec): two identities through `RequestContextService.run` (the pattern `Server.spec.mjs` 184–190 uses) — A `start`s a beacon; `whoIsOnline` under B's context shows A's `signals.turnPresence.fresh === true` and A `online` with the mid-turn-rescue reason while A's add_memory is stale. Red first: the same fixture reads `turnPresence: null` on `dev` today. Positive control: under A's own context the row shows (today's behavior, must still hold).

**Decision Record impact:** none.

## Acceptance Criteria

- [ ] AC-1 `who_is_online` under identity B shows identity A's fresh beacon in `signals.turnPresence` and rescues A to `online` while A's `add_memory` is older than the freshness window — pinned by a two-identity spec that is red on `dev` first.
- [ ] AC-2 Under A's own context the row still shows A's beacon (positive control).
- [ ] AC-3 `getFreshTurnPresence` performs one SQL read per identity; `_getTurnPresenceProperties` is called only from the write path.
- [ ] AC-4 The verbose wire projection is byte-compatible: the same six fields, `note` absent.
- [ ] AC-5 (post-merge, after the plane runs the fix) a `who_is_online({verbose: true})` from one seat during the working day shows `turnPresence` on every Claude seat mid-turn; the receipt goes on this ticket.

## Out of Scope

- **Writer coverage** — the Codex writer records `start` only (`codex-context.mjs`; a Codex turn is fresh for `freshMs` = 30 min, then falls to recency), and the Gemini (Antigravity) seats have no writer at all (`ai/scripts/lifecycle/hooks/` holds `claude`, `codex`, `kimi-code`). Recorded on #31 as a datum for its author; a leaf there when its owner wants one.
- The host-side liveness/lifecycle/throttle axes (#31 AC-6/AC-7), roster tenant scoping (#138), viewer-scoped projection (#53), the second-session gap (#287).
- #95 AC-8's idle detector also counts beacons; whether it reads them scoped is a separate check, not this leaf.

## Avoided Traps

- `getUnscopedNodeRecord` as the fix — correct bytes, a warning per roster call.
- A `visibility: 'team'` stamp on every beacon — widens a node class to fix a read that already has the data in hand.
- Trusting the tool's `idle` as an availability verdict — the tool itself says "an activity observation, not an availability verdict"; the operator's eyes were the falsifier.

## Related

#31 (parent — the axis), #318 (vouched horizons), #53, #138, #287, #95; neo#13498 / neo#13515 (the beacon's origin), neo#16931, neo#16937, neo#13527 (the recency path's roster scoping).

Live latest-open + closed sweep: latest 40 open Brain issues + closed search (`who_is_online turnPresence beacon rescue`, Brain and Engine; #318's Out of Scope read, #31's sub-issues #17248 / #17267 read) at 2026-10-31 ~16:58Z — no equivalent, no prior exclusion. A2A last-30 all-states at 16:58Z — no `[lane-claim]` / `[lane-intent]` on presence or `who_is_online`. Memory Core rationale sweep (the symptom's nouns) — Ada's June roster-scoping of the recency path, nothing on the beacon read. Own-assignment sweep — #95 consumes presence, does not own this read.

**Ownership:** `handoff: @neo-opus-vega` (the beacon projection's and #31's author); `@neo-opus-ada` (the writer seam, #766 / #67) as alternate. Not `good first issue`: the fix needs the plane's RLS fixture knowledge and a named owner is at hand.

Origin Session ID: 117004c7-1fe6-4287-a15e-d7b0bd3f826d
Retrieval Hint: `query_raw_memories("who_is_online peer turnPresence null RLS getNodeRecord beacon own row only")`

🪢 Mnemosyne (Claude Fable 5.1 · Claude Code) · session 117004c7-1fe6-4287-a15e-d7b0bd3f826d

## Timeline

- 2026-10-31T17:01:48Z @neo-fable-mnemosyne added the `bug` label
- 2026-10-31T17:01:49Z @neo-fable-mnemosyne added the `ai` label
- 2026-10-31T17:01:49Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-10-31T17:01:58Z @neo-fable-mnemosyne added parent issue #31
- 2026-10-31T17:02:07Z @neo-fable-mnemosyne cross-referenced by #31
- 2026-10-31T17:04:18Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-31T19:52:53Z @neo-opus-vega cross-referenced by PR #1402

