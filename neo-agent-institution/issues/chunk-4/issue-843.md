---
id: 843
title: A PAT update must not restart the Fleet Manager
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-30T22:40:38Z'
updatedAt: '2026-10-31T17:43:22Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/843'
author: neo-opus-ada
commentsCount: 0
parentIssue: 424
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
# A PAT update must not restart the Fleet Manager

## Context

Row 5's installed walk on 2026-10-30 (receipt [#516 6534998715](https://github.com/neomjs/neo-agent-institution/issues/516#issuecomment-6534998715)) measured two whole-app relaunches caused by credential changes:

- Admitting a fresh PAT through "Plane admission (operator PAT)" → **Admit**: SIGINT 22:10:41Z → relaunch → recovery 22:11:19Z.
- The recovery the product itself offers after a refused PAT ("The plane refused that PAT. Connect again with a current one." → **Connect**): SIGINT 22:36:23Z → recovery 22:37:06Z, then the #841 roster-timeout transient until ~22:38Z.

Each relaunch takes about 30 s, even on an M5 Max (operator). Every running seat goes through a continuity handoff (7 tonight), open views are lost, and the form never says it will restart. **Operator requirement (2026-10-30): no FM restart for any PAT update; a label or confirm is not enough.**

## The Problem

`harness/planeConfig.mjs` `createPlaneBroker().attach()` prompts the credential, verifies it with the plane, writes `plane.json` (bearer through safeStorage) and calls `relaunch()`. `harness/main.mjs` wires that as `app.relaunch(); app.quit()`. Boot is the only path that admits a plane credential ("the launched PAT"). Every credential change therefore costs a full relaunch, including recovery from an expired or revoked PAT, which happens exactly when the operator is already disrupted.

## The Architectural Reality

- The main process keeps PAT custody; the renderer never holds it (`planeConfig.mjs` header). The Brain child receives the credential at spawn.
- `verify()` already asks at runtime whether the plane still admits the launched PAT (`PlaneCredentialCheck`, `planeCredentialRecheckInterval` 300 s). Adopting a *new* PAT at runtime is the missing half.
- The cockpit already renders a plane outage and its recovery without a restart (H1 on #516).
- *(drift probe, 2026-10-31)* ADR 0034 item 8: plane configuration never routes through `FleetControlBridge` / `FLEET_WIRE_METHODS` ("the shell's boot input, not a fleet verb"), and its transitional seam hands the bearer to the fleet child in the child's environment.
- *(drift probe, 2026-10-31)* The Brain arms its plane clients once: `devFleetServer.mjs` resolves its client "ONCE at boot, BEFORE viewer binding", and `fleetServer.mjs` arms through `assertFleetPlaneBearerClass`. The Brain child is a `node:child_process` spawn (`harness/brain.mjs`), so no main↔child channel exists besides `FleetControlBridge`.

## The Fix

*(Sharpened 2026-10-31 by a pre-implementation drift probe. Step 1 first sent the bearer over "the existing authenticated main↔Brain channel", which is `FleetControlBridge`, the route ADR 0034 item 8 forbids.)*

Live rebind:
1. Main opens a dedicated credential pipe to the Brain child at spawn: an extra stdio fd, named in the child's environment. It carries the plane bearer at boot, and again after `attach()` (and the Connect recovery that reuses it) has verified a new PAT with the plane. The pipe is boot input, never a fleet verb. ADR 0034 item 8's transitional seam is amended in the same change.
2. The Brain's plane clients adopt a bearer from the pipe only after the plane proves its subject is the bound identity. They hold it as their own state and never write it into `AiConfig` (ADR-0019 B4).
3. Main persists `plane.json` only after the Brain confirms.
4. No `app.relaunch()`. The window, renderer state and seats stay up, and the spine shows a short "verifying PAT" state.
5. If the plane refuses the new PAT, the old binding stays active and the card shows the refusal in the existing vocabulary.

Rejected routes:
- A `FleetControlBridge` verb: ruled out by item 8's ownership boundary.
- Re-reading `fleet.planeBearerFile` at use: it needs the plaintext bearer at rest on the desktop, which the sealed `plane-bearer.bin` exists to avoid.
- Waiting for #17's pure-client FM: #17 is open and has been unassigned since 2026-10-14.

**Open question for the operator:** the client binds its viewer identity at boot. Should a PAT whose subject differs from the bound identity also rebind live, or keep today's relaunch? Same-subject rotation (expired or revoked PAT) is live either way.

## Acceptance Criteria

- [ ] AC-1 Admitting a PAT through the plane admission form updates the running FM with no relaunch: no SIGINT or relaunch in `main.log`, the renderer session survives, and seats are untouched. The cockpit returns to `live` with the new identity verified.
- [ ] AC-2 After "pat refused", Connect with a current PAT returns the cockpit to `live` with no relaunch.
- [ ] AC-3 A PAT the plane refuses at admission leaves the current binding active and shows the refusal on the card, with no relaunch and no lost state.
- [ ] AC-4 The bearer never reaches the renderer (the existing custody invariant), covered by a spec.
- [ ] AC-5 (installed) Row 5's H3 re-walk receipt shows refuse → Connect → `live` with no relaunch.

## Out of Scope

- The roster read timing out after a cold launch (#841).
- PAT-refusal detection latency: the cockpit's 300 s re-check stacks on the plane's 300 s cache, so detection can take up to ~10 min. That's a separate leaf.

## Avoided Traps

- A warning label or confirm dialog: rejected by the operator as insufficient.
- Restarting only the Brain child: still a restart, and still disrupts seats.

## Related

#424 (row 5's outcome), #516 (step list and receipts), #828 / PR #832 (refusal reaches the spine), #841, #211 (attach from first-run config), #17 (pure-client FM).

Live latest-open + closed sweep: checked the latest 20 open issues plus closed/all searches ("relaunch PAT", "restart credential attach"; read #211 and #727, no prior decision on relaunch-after-attach) at 2026-10-30T22:4xZ. A2A sweep (last 15, through 22:25Z): no claim on this scope. Memory Core sweep and own-assignment sweep: no equivalent. Re-swept 2026-10-31 (Memory Core + issue search for bearer rotation without restart): no prior decision.

Retrieval Hint: "FM PAT update relaunch planeConfig attach live rebind no restart"

Origin Session ID: 2e07d6d3-af03-4b39-9254-f979cc291d2f


## Timeline

- 2026-10-30T22:40:39Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-30T22:40:40Z @neo-opus-ada added the `enhancement` label
- 2026-10-30T22:40:40Z @neo-opus-ada added the `agent-os` label
- 2026-10-30T22:40:41Z @neo-opus-ada added the `ai` label
- 2026-10-30T22:44:48Z @neo-opus-ada cross-referenced by #844
- 2026-10-30T22:44:55Z @neo-opus-ada cross-referenced by #845
- 2026-10-30T22:45:37Z @neo-opus-ada added parent issue #424
- 2026-10-30T23:49:38Z @neo-opus-ada cross-referenced by #847
- 2026-10-31T00:05:13Z @neo-opus-ada cross-referenced by #849
- 2026-10-31T14:50:56Z @neo-gemini-maryam cross-referenced by PR #850
- 2026-10-31T18:39:10Z @neo-opus-ada cross-referenced by PR #851

