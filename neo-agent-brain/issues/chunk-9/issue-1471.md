---
id: 1471
title: 'The roster names the Fleet''s start gate once, not only per seat'
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-04T23:54:41Z'
updatedAt: '2026-11-05T00:05:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1471'
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
blockedBy: []
blocking: []
---
# The roster names the Fleet's start gate once, not only per seat

## Context

neomjs/neo-agent-institution#841 AC-3 asks: "While the fleet binds, the state line names the boot phase it is in." The fleet already says it, but only on each roster row. #1260 stamps every row's `launchRefusal` with `startGateRefusal()`, "the Fleet is verifying its runtime: Start opens when it finishes", while runtime verify or recovery runs. The cockpit shows that per seat, on the card's Start and the detail's runtime line. Its state line has no fleet-level fact to name.

#1468 makes the roster answer during boot, so the gate's sentence now reaches the cockpit while the fleet binds.

## The Problem

`createFleetCockpitStatus` returns `{sources, capabilities, rows, events}`, and a seat's own refusal outranks the gate on that seat's row. So the renderer cannot tell "this Fleet refuses every Start until it has booted" from "every seat happens to refuse". To name the boot phase once, the roster must carry the gate itself.

## The Architectural Reality

- `ai/services/fleet/FleetControlBridge.mjs` `fleetRoster()` reads `startGate = manager.getLifecycleService().startGateRefusal()` and uses it only as each row's fallback `launchRefusal`.
- `ai/services/fleet/FleetLifecycleService.mjs` `startGateRefusal()` returns the gate's words, or `null` when Start may run.
- `ai/services/fleet/fleetCockpitStatus.mjs` `createFleetCockpitStatus()` is the DTO's one constructor.
- Design authority: `startGateRefusal`'s docblock, "A booting packaged Fleet answers its wire before verification and recovery settle, so the roster says why Start stays closed until they do." The roster is meant to say it, and today says it only through each seat.

## The Fix

`FleetLifecycleService#startGateRefusal` becomes `startGate()` and returns `{state, reason}`. `state` is `'booting'` while verification or recovery runs, so Start opens by itself; it is `'refused'` for a gate that holds until something changes. A sentence alone cannot tell the two apart, and the cockpit needs that for its line and for how often it re-reads. `fleetRoster` reads the gate once: rows keep its reason as their `launchRefusal`, and `createFleetCockpitStatus` emits the whole gate at the DTO's top level, `null` when Start may run.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `fleetRoster` DTO `startGate` (new, additive) | `FleetLifecycleService#startGate` | `{state: 'booting'\|'refused', reason}` while the Fleet refuses every Start, else `null` | Absent on an older Brain: a reader treats absent as `null` | `createFleetCockpitStatus` JSDoc | AC-1 |
| `FleetLifecycleService#startGate` (replaces `startGateRefusal`, Brain-internal) | `start()`'s fleet-wide gates, in its order | The gate's state and its reason in the closed Start's words | `null` when Start may run | Its JSDoc | AC-3 |

## Acceptance Criteria

- [ ] AC-1 (unit, `FleetControlBridge.spec.mjs`): while the lifecycle's gate is shut, `fleetRoster()` carries it as `startGate`, also when a seat's own refusal stands on that seat's row. Once the gate opens, `startGate` is `null`.
- [ ] AC-2: the rows' `launchRefusal` is unchanged, and #1260's roster arm stays green.
- [ ] AC-3 (unit, `FleetLifecycleService.spec.mjs`): the gate reads `booting` while verification or recovery runs, and `refused` for a failed verification, an unavailable continuity and frozen custody.

## Out of Scope

- The cockpit's state line, and how often it re-reads the roster while the gate is shut (neomjs/neo-agent-institution#841 AC-3).
- The fleet's individual boot phases (bind, plane init, verify, recovery). The gate's sentence is the phase the cockpit can act on.

## Related

neomjs/neo-agent-institution#841 (the outcome), #1260 (the per-seat gate), #1468 (the roster answers during boot).

Live latest-open + closed sweep: latest 20 open Brain issues at 2026-11-04T23:53:51Z; all-state search "start gate roster OR startGateRefusal OR verifying its runtime" (#1260 read: its Fix, Out of Scope and Avoided Traps exclude a held Start and `launchRefusalOf`, not a fleet-level field); no equivalent. A2A sweep: inbox read at 23:52Z, no claim on the roster or the gate. MC sweep: "fleet start gate verifying its runtime cockpit state line roster startGate top level boot phase", Sophie's 2026-10-16 diagnosis of the first roster read (fixed by neomjs/neo-agent-institution#707), no prior decision on a fleet-level gate. Own-assignment: #1468 (in review), not this field.

Origin Session ID: 2171b2cd-e447-4cac-8b9f-ccadc6e20db5
Retrieval Hint: "roster DTO startGate fleet-level start gate verifying its runtime cockpit state line boot"


## Timeline

- 2026-11-04T23:54:41Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-04T23:54:43Z @neo-opus-ada added the `enhancement` label
- 2026-11-04T23:54:43Z @neo-opus-ada added the `ai` label
- 2026-11-04T23:54:43Z @neo-opus-ada added the `agent-os` label
- 2026-11-04T23:57:35Z @neo-opus-ada cross-referenced by PR #1472
- 2026-11-05T00:05:23Z @neo-opus-ada referenced in commit `84f927d` - "feat(fleet): the start gate says whether the Fleet is booting (#1471)

A string could not tell a gate that opens by itself (runtime verification or recovery still running) from one that holds until something changes, and the cockpit needs that for its line and its re-read. FleetLifecycleService#startGate replaces startGateRefusal and returns {state: 'booting'|'refused', reason}; rows keep the reason as their launchRefusal, and the roster DTO carries the whole gate."

