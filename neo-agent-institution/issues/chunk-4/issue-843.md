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
updatedAt: '2026-11-04T12:54:06Z'
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

- Admitting a fresh PAT through the plane card → **Connect**: SIGINT 22:10:41Z → relaunch → recovery 22:11:19Z. *(Corrected 2026-11-04: this line first said "Plane admission (operator PAT)" → Admit. Admit is `connectTenant`, which never relaunches; only `attach()` and the seat-root move call `app.relaunch()` in `harness/`.)*
- The recovery the product itself offers after a refused PAT ("The plane refused that PAT. Connect again with a current one." → **Connect**): SIGINT 22:36:23Z → recovery 22:37:06Z, then the #841 roster-timeout transient until ~22:38Z.

Each relaunch takes about 30 s, even on an M5 Max (operator). Every running seat goes through a continuity handoff (7 tonight), open views are lost, and the form never says it will restart. **Operator requirement (2026-10-30): no FM restart for any PAT update; a label or confirm is not enough.**

## The Problem

`harness/planeConfig.mjs` `createPlaneBroker().attach()` prompts the credential, verifies it with the plane, writes `plane.json` (bearer through safeStorage) and calls `relaunch()`. `harness/main.mjs` wires that as `app.relaunch(); app.quit()`. Boot is the only path that admits a plane credential ("the launched PAT"). Every credential change therefore costs a full relaunch, including recovery from an expired or revoked PAT, which happens exactly when the operator is already disrupted.

## The Architectural Reality

- The main process keeps PAT custody; the renderer never holds it (`planeConfig.mjs` header). The Brain child receives the credential at spawn.
- `verify()` already asks at runtime whether the plane still admits the launched PAT (`PlaneCredentialCheck`, every 45 s since #845). Adopting a *new* PAT at runtime is the missing half.
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
5. If the plane refuses the new PAT, or names another account than the one this FM launched with, the old binding stays active and the card shows the refusal. For another account, the refusal names both.

Rejected routes:
- A `FleetControlBridge` verb: ruled out by item 8's ownership boundary.
- Re-reading `fleet.planeBearerFile` at use: it needs the plaintext bearer at rest on the desktop, which the sealed `plane-bearer.bin` exists to avoid.
- Waiting for #17's pure-client FM: #17 is open and has been unassigned since 2026-10-14.

*(Decided 2026-10-31: a PAT for another account is refused, never adopted. This replaces the open question on it.)* The plane names a PAT's subject as a canonical `@login` (`list_permissions` inside `probePlaneCredential`), and it admits agent accounts too: this seat's own plane session resolves `@neo-opus-ada` through that call. The operator handles a PAT for every agent account, so another subject at this form is most likely a wrong paste. Today `attach()` writes whatever subject the plane names and relaunches, so after a wrong paste the fleet child comes back up as that agent.

## Delivery

*(Intake 2026-11-03: drift probe since the 10-31 sharpening.)* The prescription stands. Since then only #849 / PR #850 touched these files: an in-process Retry for a refused boot, while the Brain child is down. It amended ADR 0034 §2.3 item 8, so this ticket's amendment builds on that text.

Two repositories, one PR each, in order:
1. neomjs/neo-agent-brain#1462: the fleet server adopts a bearer from the pipe without a restart. It builds a candidate client and proves it before the swap, because `planeMailboxClient#init` closes its prior life first. Done: PR neomjs/neo-agent-brain#1463, merged 2026-11-03 as `be1bde55`.
2. This ticket's PR: the pipe at spawn, `attach()` and Connect over it, persistence after the Brain confirms, the spine state, and the ADR 0034 amendment.

*(Corrected 2026-11-03: no separate Brain pin move.)* CI installs the Brain at `dev` (`npm run resolve-org-dev`, per #646). A packaged cut takes the Brain runtime root's own revision and checks it against `origin/dev` (`harness/pack.mjs`, `resolveCutRevisions` and `assessBrainFreshness`). Both therefore already reach `be1bde55`. The `package.json` SHA decides only what a plain `npm ci` installs, so this ticket's PR moves it in passing, as #854 did.

## Implementation notes

*(2026-11-03, from the Brain side as neomjs/neo-agent-brain#1463 has it at `5118008f`, in review.)*
- **The pipe:** the spawn in `harness/brain.mjs` adds a fourth `stdio` entry, `'pipe'`, and sets `NEO_FLEET_PLANE_CREDENTIAL_FD=3`. Each frame is one JSON line, `{"bearer": "…", "bearerClass": "<authSource>"}`; the first is the boot bearer. The child answers every frame in order: `{"state": "adopted", "identity": "@x"}`, or `{"state": "refused", "reason": "…"}`, plus `resolvedIdentity` and `expectedIdentity` for another account. A refused boot frame is answered before the child exits 1.
- **The env fragment:** with the pipe, `planeEnvFragment` keeps `NEO_FLEET_PLANE_BASE` and `NEO_AGENT_IDENTITY` and drops the bearer and its class. A declared `NEO_FLEET_PLANE_BEARER` or `NEO_FLEET_PLANE_BEARER_FILE` beside the pipe refuses the boot, even an unreadable one.
- **`attach()`:** when the running child was launched with the pipe for the same plane base, it probes as today, writes a rotation frame and awaits the answer. Adopted: `writePlaneConfig`, then `{ok: true, relaunching: false}`. Refused: `plane.json` stays untouched and the refusal returns, naming both accounts from `resolvedIdentity` / `expectedIdentity`. A host-mode launch or a different plane base keeps today's relaunch.
- **Closing the pipe** ends rotations only; the child keeps its bound bearer.

*(2026-11-04, shell side, from the implementation read.)*
- **Spawn:** `startBrainChild({credentialPipe: true})` owns the fourth `stdio` entry and names it in the child's env. Main's end is `harness/planeCredentialPipe.mjs`: answers are matched to frames in order, and a pipe that closes answers every pending frame as refused. A respawn pipes the current record, the rotated one included.
- **The fragment blanks** `NEO_FLEET_PLANE_BEARER`, `_FILE`, `_CLASS` and both admission variables: an env fragment can override an inherited value but never remove it, and the Brain refuses a boot that declares a bearer beside the pipe.
- **A refused boot:** when no child is running (the boot settled down with `plane-credential-refused` or `plane-identity-changed`), there is no working binding to keep, so `attach()` stores the record and runs the boot again in this process (`retryBrainBoot`). That is also "pat refused" → Connect, at boot.
- **Another account** is refused whenever this shell launched from a record for the same plane base, in every path, not only in a rotation. The `plane-identity-changed` banner therefore stops saying "Connect again to confirm it".
- **The cockpit:** after an adopted rotation, the card hands off to the viewport, which re-drives the cockpit (the `onConnectPlane` precedent). A reconnect asks `verifyPlane()` again while a credential cause stands, and `verifyPlane()` now probes the rotated record. Without that, "pat refused" would clear only at the 300 s re-check.
- **"Verifying PAT"** moves from the spine (step 4) to the card's status line, where the operator acted and waits for the reply. The spine keeps naming the fault until the shell's verdict clears it.

## Acceptance Criteria

- [ ] AC-1 Admitting a PAT through the plane admission form updates the running FM with no relaunch: no SIGINT or relaunch in `main.log`, the renderer session survives, and seats are untouched. The cockpit returns to `live`, with the new PAT verified for the same identity.
- [ ] AC-2 After "pat refused", Connect with a current PAT returns the cockpit to `live` with no relaunch.
- [ ] AC-3 A PAT the plane refuses at admission, or admits for another account, leaves the current binding and `plane.json` unchanged and shows the refusal on the card, naming both accounts in the second case. No relaunch and no lost state.
- [ ] AC-4 The bearer never reaches the renderer (the existing custody invariant), covered by a spec.
- [ ] AC-5 (installed) Row 5's H3 re-walk receipt shows refuse → Connect → `live` with no relaunch.

## Out of Scope

- The roster read timing out after a cold launch (#841).
- PAT-refusal detection latency: #845 bounded it at 360 s (PR #851).
- Changing the account the FM runs as. That is not a PAT update; it gets its own ticket if it is ever needed.

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
- 2026-10-31T20:54:52Z @neo-gemini-maryam cross-referenced by PR #854
- 2026-11-03T08:17:14Z @neo-opus-ada cross-referenced by #864
- 2026-11-03T08:43:45Z @neo-opus-ada cross-referenced by #1462
- 2026-11-03T17:51:58Z @neo-opus-ada cross-referenced by PR #1463
- 2026-11-04T13:23:04Z @neo-opus-ada cross-referenced by PR #870

