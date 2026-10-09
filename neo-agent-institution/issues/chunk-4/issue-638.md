---
id: 638
title: Row 2's fixture walkthrough reads the runtime root's real registry
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-grace
createdAt: '2026-10-09T06:53:26Z'
updatedAt: '2026-10-09T12:12:50Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/638'
author: neo-opus-grace
commentsCount: 0
parentIssue: 477
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-09T12:12:50Z'
---
# Row 2's fixture walkthrough reads the runtime root's real registry

## Context

`e2e/agentos/CockpitStateWalkthroughNL.spec.mjs` (#498, mine, PR #494) provokes six states on a mounted cockpit against an in-process Fleet bridge, and expects an empty fleet in each: `Fleet · 0 agents` with the `Add your first agent` CTA. On 2026-10-09, clean dev `32627ab` with `NEO_AGENTOS_RUNTIME_ROOT` set to a seat's working Brain checkout, it fails at the first state that reads the roster:

```
- "emptyCta": "Add your first agent",
+ "emptyCta": null,
- "title": "Fleet · 0 agents",
+ "title": "Fleet · 12 agents",
```

Twelve is this machine's real registry; the tests' sample fleet has eleven. Sophie's #601 baseline shows the same red in her seat. It is the only red of the 77 Neural Link specs on clean dev here.

Sweeps:
- **Live:** the latest 20 open Institution issues at 2026-10-09T06:53Z, and `gh search issues CockpitStateWalkthrough`; no equivalent (#498 is the closed original).
- **A2A:** the latest 30 messages; no claim. Sophie's baseline note reported the symptom.
- **Memory Core:** my own walkthrough work on #498; no decision on registry isolation.
- **Own assignments:** #635, #633, #616, #610, #589, #490, #414, #11; none overlaps.

## The Problem

The spec reads whatever registry the runtime root holds. That is empty in a scratch Brain checkout at the pin, which is how it was written and passed. In any seat's working Brain it is the team's real roster, so the walkthrough goes red for everyone who runs the battery from a seat, and its "empty fleet" receipt was never guaranteed by the test.

## The Architectural Reality

- The spec starts the Brain's `fleetBridgeServer` in process (`authenticatedFleetHarness.startRealFleetServer`). `FleetControlBridge#fleetRoster` lists agents through `getRegistry().listAgents()`.
- `FleetControlBridge` documents `registry` and `manager` as plain injectable fields: "inject a stub in tests".
- `withWalkthroughFleet` already saves and restores `bridge.activitySource` around the run.

## The Fix

In `withWalkthroughFleet`, inject a registry that answers `listAgents()` with no agents and delegates every other call, bound, to the real registry. Restore the original in the same `finally` that restores `activitySource`. This touches only the spec.

## Acceptance Criteria

- [ ] The walkthrough passes with `NEO_AGENTOS_RUNTIME_ROOT` pointing at a Brain whose registry holds agents, here a seat's working checkout with 12.
- [ ] The injected registry is restored on every exit, so no later spec in the run reads it.
- [ ] A control: without the injection, the same run reads the real count and fails.

## Out of Scope

- The census words and the six provoked states (#498).
- The Brain's registry seams themselves.

## Related

#498 · #477 · #601 (where the red was reported) · #11

Origin Session ID: e76b2469-377c-4fec-85a7-4c47b10269b9
Retrieval Hint: "CockpitStateWalkthroughNL real registry 12 agents runtime root isolation listAgents"


## Timeline

- 2026-10-09T06:53:27Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-09T06:53:27Z @neo-opus-grace added the `bug` label
- 2026-10-09T06:53:28Z @neo-opus-grace added the `ai` label
- 2026-10-09T06:53:28Z @neo-opus-grace added the `testing` label
- 2026-10-09T06:53:35Z @neo-opus-grace added parent issue #477
- 2026-10-09T07:03:12Z @neo-opus-grace cross-referenced by PR #639
- 2026-10-09T12:12:50Z @tobiu referenced in commit `fbe3f78` - "fix(e2e): row 2's walkthrough reads an empty fleet, never the runtime root's real registry (#638) (#639)"
- 2026-10-09T12:12:50Z @tobiu closed this issue

