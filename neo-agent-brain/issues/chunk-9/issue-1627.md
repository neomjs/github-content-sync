---
id: 1627
title: The fleet sets operator presence and serves it to the cockpit
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-12T02:56:36Z'
updatedAt: '2026-11-12T02:56:36Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1627'
author: neo-opus-ada
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 1625 Whether the operator is away is plane state every seat can read'
blocking: []
---
# The fleet sets operator presence and serves it to the cockpit

## Context

Operator, 2026-11-12: a Nightshift toggle in the FM product bar (`neomjs/neo-agent-institution#929`). The plane record and its human-only write are #1625 (PR #1626). The Institution app reaches the fleet only through the wire contract: `apps/agentos/fleet/installFleetBridge.mjs` builds its proxy map from `FLEET_WIRE_METHODS`. So the toggle needs a fleet verb to write the mode and a cockpit DTO field to read it. #929 as filed put "writes through the fleet server… the Fleet then sends one broadcast" in the Institution ticket, but that half is Brain code. This ticket takes it.

## The Problem

- `FLEET_WIRE_METHODS` (`src/fleet/contract/wire.mjs:11`) has no presence verb, so the cockpit cannot write the mode.
- The cockpit DTO (`createFleetCockpitStatus`, `ai/services/fleet/fleetCockpitStatus.mjs:50`) carries no mode, so the toggle cannot show the plane's state on load or after a change made elsewhere.
- Peers must learn of a change once, from the operator identity, without a wake.

## The Architectural Reality

- **Operator writes ride bridge verbs over a writer seam the boot installs.** `FleetControlBridge#composeOperatorMessage` (`FleetControlBridge.mjs:1526`) calls `composeWriter.addMessage`, and `wireOperatorComposeWriter.mjs` installs the writer. Plane mode binds `planeClient.addMessage`; local mode binds `MailboxService.addMessage` (`devFleetServer.mjs:530`, `:558`). The sender is never a parameter: the plane stamps the transport-bound identity, the FM's proven viewer.
- **Every verb is classified twice** in `ai/services/fleet/fleetServerPolicy.mjs`: `FLEET_S1_METHOD_POLICY` (slice) and `FLEET_METHOD_SCOPE_CLASSES` (`lifecycle-write` needs the owner principal). An unclassified verb refuses fail-closed.
- **The roster already reads the record.** It reads `who_is_online` verbose every tick (`planeWhoIsOnlineReader.mjs`), and after #1625 that answer carries `operatorPresence`. Today `FleetManager#fleetPresenceStatus` (`FleetManager.mjs:891`) → `readFleetPresenceSnapshot` (`fleetPresenceStateAdapter.mjs:228`) keeps only the per-seat states.
- **`set_operator_presence` answers `{operatorPresence, changed}`.** `changed` is the once-per-change signal, so the fleet needs no read first.

## The Fix

1. `setOperatorPresence` joins `FLEET_WIRE_METHODS`, classified `awaiting-s4` / `lifecycle-write` like `composeOperatorMessage`.
2. `FleetControlBridge#setOperatorPresence({mode, until})` calls the writer's `setOperatorPresence`: plane mode `planeClient.callTool('set_operator_presence', …)`, local mode `OperatorPresenceService`. On `changed: true` it sends one `AGENT:*` message through the same writer's `addMessage`, with subject `[operator-presence] away until <HH:MM> UTC` or `[operator-presence] present`, `wakeSuppressed: true` and `taggedConcepts: ['operator-presence']`.
   - It answers the plane's record, `changed`, and the broadcast's `messageId` or failure reason. The state write stands either way.
   - An unwired writer answers `not-wired`.
3. `wireOperatorComposeWriter` installs the presence primitive beside `addMessage`, in both boot branches.
4. The presence report keeps the payload's `operatorPresence`, and `fleetRoster` hands it to `createFleetCockpitStatus` as a top-level `operatorPresence`. A presence read that did not answer gives `null`, so the toggle shows unknown, never present.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `setOperatorPresence` fleet verb (new) | operator 2026-11-12, `neomjs/neo-agent-institution#929` | sets the mode through the proven plane identity; one non-wake broadcast per change | unwired writer → `not-wired`; a plane refusal → sanitized error | JSDoc + `wire.mjs` | unit: verb → writer, one broadcast on `changed`, none otherwise, policy rows |
| cockpit DTO `operatorPresence` (new) | #1625 | the plane's record from the roster's `who_is_online` read | unread → `null` | `fleetCockpitStatus.mjs` JSDoc | unit: the DTO carries it; `null` when the presence read failed |

## Decision Record impact

none

## Acceptance Criteria

- [ ] `setOperatorPresence` is a wire verb classified `lifecycle-write`, and it writes the mode through the plane's `set_operator_presence`.
- [ ] Exactly one `[operator-presence]` non-wake `AGENT:*` message goes out per write that answers `changed: true`, and none otherwise.
- [ ] The roster DTO carries `operatorPresence`, or `null` when the presence read did not answer.
- [ ] Specs cover the verb, the once-per-change rule, the policy rows and the DTO field.

## Out of Scope

- The toggle UI (`neomjs/neo-agent-institution#929`).
- The plane record (#1625).
- #95's wake policy and `neomjs/neo-agent-institution#842`.

## Avoided Traps

- **A separate read verb:** one more plane call per poll, while the roster's existing `who_is_online` read already carries the record.
- **The broadcast inside the plane:** no plane service sends mail on a caller's behalf today, and the fleet's writer already owns operator-authored mail.
- **Deriving "changed" in the fleet by reading first:** it races and costs a call, and the plane already answers it.

## Related

#1625 (blocks this), `neomjs/neo-agent-institution#929` (blocked by this), #95, `neomjs/neo-agent-institution#842`, #1508 (the FM MVP epic; this is an operator feature outside its define/start/observe scope).

Live latest-open + closed sweep: Brain 155 open (red band; latest 20 read, `#1625`..`#1512`) plus closed searches `operator presence` and `nightshift` at 2026-11-12T02:55Z. The closed hits are roster-read timing (#1622, #1586, #1468), none on this surface; no owner, so filed standalone.
A2A in-flight claim sweep: latest 30 messages, all states, at 02:55Z; no claim on this surface.
MC sweep: `cockpit cannot set the operator's away mode; Nightshift toggle needs a fleet verb`, 5 results (the 11-12 design turns, per-seat nightshift announcements); no prior decision.
Own-assignment sweep: 6 open; #1625 is the plane half, #1508 the MVP epic (above).
Structure map: `npm run ai:structure-map -- --files --loc` exit 0; owner `ai/services/fleet` (siblings `FleetControlBridge.mjs`, `fleetServerPolicy.mjs`, `wireOperatorComposeWriter.mjs`); no new file.

Retrieval Hint: "fleet setOperatorPresence verb, operator-presence broadcast once per change, cockpit DTO operatorPresence, Nightshift toggle Brain half"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-12T02:56:36Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-12T02:56:38Z @neo-opus-ada added the `enhancement` label
- 2026-11-12T02:56:38Z @neo-opus-ada added the `ai` label
- 2026-11-12T02:56:38Z @neo-opus-ada added the `agent-os` label
- 2026-11-12T02:56:47Z @neo-opus-ada marked this issue as being blocked by #1625

