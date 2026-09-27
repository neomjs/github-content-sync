---
id: 565
title: An explicit launchOwner at defineAgent records launchOwnerSince
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-27T09:12:57Z'
updatedAt: '2026-09-27T09:47:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/565'
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
closedAt: '2026-09-27T09:47:42Z'
---
# An explicit launchOwner at defineAgent records launchOwnerSince

## Context

Found while the swarm prepared the Fleet Manager roster pilot (operator direction 2026-09-27: the installed cockpit's roster gets the six running peers through Add agent; Euclid's enrollment-boundary question, Ada's recommendation, both on A2A 09:01–09:10Z). Observed at Brain dev `4bc885b`: `launchRefusalOf` (`src/fleet/contract/launchAuthority.mjs`) refuses a start only for `launchOwner === 'external' && launchOwnerSince`; `FleetManager.assertStartPermitted` (`ai/services/fleet/FleetManager.mjs:354`) and `spawnPermitted` (`ai/services/fleet/startAgentProvisioned.mjs:49`) consult it; `FleetRegistryService#defineAgent` (`ai/services/fleet/FleetRegistryService.mjs:275–345`) stores `launchOwner` and never `launchOwnerSince`, which only `setLaunchOwner` (`:552`) writes. So a row defined with `launchOwner: 'external'`, explicitly or by default, answers `null` to the refusal — observation from the predicate and both callers, not inference.

## The Problem

The registry's doctrine paragraph (`FleetRegistryService.mjs:195–202`) says a row without an ownership fact reads `external` and "the fleet refuses to start it", but the predicate keys on the timestamp of an ownership ACT, and creation records none. "A peer that already runs in its own harness" therefore cannot be expressed atomically: the only way to make the fleet refuse its start is `defineAgent` followed by `releaseAgent`, two writes with a startable window between them. Every Add agent submission today stamps `launchOwner: 'fleet'` anyway (Institution `apps/agentos/util/AddAgentFlow.mjs:149`). For the pilot that means six rows whose start authority is wrong from birth on the wire (`startAgent` is a wire verb), even though the cockpit card's runtime gate leaves Start disabled for a never-launched seat (#382's table: never launched, its own harness → not eligible).

## The Architectural Reality

- #375 made launch ownership a registry fact; #382 (PR #383) made an explicit release start authority through `launchOwnerSince` and left the never-acted default timestamp-less on purpose — the predicate's JSDoc: "a definition that never had an ownership act answers `null`, so its process record stays its only start gate". That case stays.
- An explicitly passed `launchOwner` at creation IS an ownership act: the caller chose. `setLaunchOwner` already treats its write as one (`launchOwnerSince: now`); `defineAgent` does not.
- Callers: `FleetControlBridge#defineAgent` (`:267`, the shell's Add agent; the shell's `projectPublicAgentIntent` forwards `launchOwner` as a public field), `ai/scripts/fleet/onboardPeer.mjs:743` (passes no `launchOwner`, the default path, unaffected), the specs.
- `toPublic` already carries `launchOwnerSince` when present; no schema change, no wire change.

## The Fix

In `FleetRegistryService#defineAgent`: when `options` carries `launchOwner` explicitly (either value), the stored definition gets `launchOwnerSince: now` in the same write; an omitted `launchOwner` keeps today's timestamp-less default. The doctrine paragraph (`:195–202`) and the option's JSDoc say that an explicit value is an ownership act. Spec arms in `test/playwright/unit/ai/services/fleet/FleetRegistryService.spec.mjs` and `FleetManager.spec.mjs`, red first.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `FleetRegistryService#defineAgent`, `launchOwner` option | `FleetRegistryService.mjs:275–345` | an explicit value also records `launchOwnerSince` in the same write | omitted → unchanged, no timestamp | option JSDoc + doctrine paragraph | registry spec arms |
| `launchRefusalOf` / `assertStartPermitted` / `spawnPermitted` | unchanged | an explicit-external row refuses a start from birth | — | predicate JSDoc unchanged | `FleetManager.spec` arm |
| wire `defineAgent` (bridge → registry) | `FleetControlBridge.mjs:267` | unchanged pass-through | — | — | bridge spec unchanged |

## Acceptance Criteria

- [ ] AC-1: `defineAgent({…, launchOwner: 'external'})` yields a row whose `launchOwnerSince` equals its `createdAt`, `launchRefusalOf(row)` is the released-seat refusal, and `FleetManager.startAgent(id)` throws with it (red first on dev).
- [ ] AC-2: `defineAgent({…, launchOwner: 'fleet'})` yields `launchOwnerSince` and a permitted start; `defineAgent({…})` without `launchOwner` yields no `launchOwnerSince` and a permitted start.
- [ ] AC-3: `adoptAgent` on an explicit-external row lifts the refusal; the existing `setLaunchOwner` arms stay green.
- [ ] AC-4: the doctrine paragraph and the option JSDoc describe the shipped rule; `onboardPeer.mjs` behaviour is unchanged.

## Out of Scope

- The Institution's Add agent ownership choice and credential-optional external submissions (the pilot's UI half, an Institution leaf).
- Where the pilot's rows live (the operator's machine registry vs a plane-side roster projection, #53 S3) — open with Emmy.
- Runtime derivation for external rows (`fleetRuntimeStatus`), unchanged since #382.

## Related

#375, #382 (PR #383), #53, #51, neomjs/neo-agent-institution#7, neomjs/neo-agent-institution#171

Live latest-open sweep: latest 20 open Brain issues at 2026-09-27T09:10Z hold no equivalent; exact sweeps for `launchOwnerSince` / `launchRefusalOf` / `adoptAgent` return only closed #375 / #382 (this ticket amends their edge, it does not reopen them). A2A in-flight sweep (last 30, any read state): no `[lane-claim]` on this surface; Euclid's and Ada's 09:04–09:10Z messages recommend exactly this contract. MC sweep: `query_raw_memories` on the symptom (a row born external is startable; a second instance of a seat running elsewhere) → Clio's #171 filing (09-19) and Grace's #375 / #382 lane, no decision against an explicit-at-creation act. Own-assignment sweep: 8 open, none overlapping. Structure map: owning folder `ai/services/fleet`, sibling precedent `setLaunchOwner`.

Decision Record impact: none.

Origin Session ID: 574ae0b8-b8d0-40d3-8cf6-1693ec48674a
Retrieval Hint: "defineAgent explicit launchOwner records launchOwnerSince; launchRefusalOf born-external row startable; FM roster pilot six peers external"

## Timeline

- 2026-09-27T09:12:57Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-27T09:12:58Z @neo-opus-vega added the `bug` label
- 2026-09-27T09:12:58Z @neo-opus-vega added the `ai` label
- 2026-09-27T09:12:58Z @neo-opus-vega added the `agent-os` label
- 2026-09-27T09:17:24Z @neo-opus-vega cross-referenced by PR #566
- 2026-09-27T09:25:03Z @neo-opus-vega referenced in commit `1199071` - "test(fleet): the born-external refusal is proved through the real registry and a fresh hydration (#565)"
- 2026-09-27T09:39:03Z @neo-opus-vega referenced in commit `a047c4c` - "test(fleet): the born-external arm hands the registry singleton its root back before the directory goes (#565)"
- 2026-09-27T09:47:42Z @tobiu referenced in commit `6696916` - "Merge pull request #566 from neomjs/vega/565-explicit-launch-owner

fix(fleet): an explicit launchOwner at defineAgent is an ownership act, recorded as launchOwnerSince in the same write (#565)"
- 2026-09-27T09:47:42Z @tobiu closed this issue
- 2026-09-27T10:01:33Z @neo-opus-ada cross-referenced by PR #281
- 2026-09-27T12:44:02Z @neo-opus-ada cross-referenced by #576
- 2026-09-27T12:59:53Z @neo-opus-ada cross-referenced by PR #577

