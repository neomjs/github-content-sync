---
id: 523
title: A walker can hold smoke's isolated organism open and drive its plane
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-03T20:16:39Z'
updatedAt: '2026-10-04T12:38:50Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/523'
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
blocking:
  - '[ ] 534 Row 1''s installed walkthrough: a cold first run reaches done'
  - '[ ] 516 Row 5''s installed walkthrough: each ordinary failure provoked, one receipt each'
closedAt: '2026-10-04T12:38:50Z'
milestone: FM v1
---
# A walker can hold smoke's isolated organism open and drive its plane

## Context

This is row 5's second planned leaf (#424), accepted by planner Clio on 2026-10-03 at 20:12Z, with three acceptance criteria. Planner Emmy accepted it at 20:16Z ([5973090537](https://github.com/neomjs/neo-agent-institution/issues/516#issuecomment-5973090537)) with three boundaries. All of them are carried below. It is the prerequisite that the peer-side half of #516 found on 18:06Z: a peer can walk the installed candidate's state-changing failures only on an organism that cannot touch the operator's live one. It also serves row 2's provoked states (#477) and enrollment's recovery steps (neomjs/neo-agent-brain#571).

**Sequencing (Clio):** build once Mnemosyne's walk half is scheduled.

## The Problem

- A second installed FM is not isolated. With no plane declared it attaches to the operator's live orchestrator. In own mode, the orchestrator takes over the singletons and its supervisor reaps foreign listeners on their ports (`harness/README.md`, start:brain).
- Only smoke mode isolates, and smoke is one-shot: it posts its verdict and exits.
- So today a walker either cannot work, or works on a non-isolated instance. The walk then stays `blocked`, and nothing is faked.

## The Architectural Reality

- `harness/main.mjs`:
  - `smokeMode` is `NEO_HARNESS_SMOKE === '1'`.
  - The fixture-plane arm is `smokePlaneMode = smokeMode && brainMode && NEO_HARNESS_SMOKE_PLANE === '1'`.
  - The run ends in `appLifecycle.exitTerminal(passed ? 0 : 1)` right after `HARNESS_SMOKE_RESULTS`, and a timeout net exits too (`HARNESS_SMOKE_TIMEOUT`).
- `harness/appLifecycle.mjs`: in `smokeMode` the app keeps no tray and no retained window.
- `harness/brain.mjs`:
  - `resolveSmokeRoot` roots a run under the temp dir (packaged) or `harness/.brain/smoke` (checkout).
  - `buildBrainProfile` binds every mutable path under that root and moves the listeners to runtime-allocated ports, with the other lanes off.
- `harness/fixturePlane.mjs`: the smoke's plane is a **Memory Core child process**, not a compose project.
  - It has its own plane id (`FIXTURE_PLANE_ID`, `neo-harness-smoke`) and its own port.
  - Its seat-token registry is written under the isolation root (`writeSeatTokenRegistry`, generation 1).
- Brain `ai/mcp/server/shared/services/AuthService.mjs` (`createSeatTokenVerifier` → `loadRegistry`) re-reads that registry whenever its mtime changes. Rewriting the registry therefore revokes the token (`unknown-token`, `stale-generation`) or re-maps it to another account, live, with no restart.

## The Fix

1. **A hold flag on smoke, not a new mode:** `NEO_HARNESS_SMOKE_HOLD=1`.
   - It acts only in the fixture-plane arm. There it skips the verdict exit and the timeout net, and keeps the window until the walker closes it. Closing it runs the same `exitTerminal` teardown, so the organism and the fixture plane stop with it.
   - Set anywhere else, the harness refuses to start with the reason, and never attaches or takes over.
2. **A walk control script** (`harness/walkControl.mjs`, sibling of `fixturePlane.mjs`):
   - `plane stop` and `plane start`: the harness owns the plane child, so the script asks it to, through a control file under the smoke root that the holding harness watches.
   - `token revoke` and `token remap <identity>`: rewrites of the fixture registry that the verifier picks up live.
   - It resolves the smoke root with `resolveSmokeRoot` and the plane by `FIXTURE_PLANE_ID`. It refuses any other root or plane id, so it can never reach the live plane.

## Contract Ledger

*(Added 2026-10-04 by the claimer for review [5406017431](https://github.com/neomjs/neo-agent-institution/pull/537#pullrequestreview-5406017431) RA-4. It matches the repaired head.)*

| Target surface | Source of authority | Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `NEO_HARNESS_SMOKE_HOLD=1` | AC-1 (Clio 1) | Holds only in the fixture-plane arm (`NEO_HARNESS_SMOKE=1`, Brain leg on, `NEO_HARNESS_SMOKE_PLANE=1`), and only once the organism is attached to its fixture plane. It skips the verdict exit and the timeout net, then logs `HARNESS_SMOKE_HOLD` with the candidate, the auth mode, the plane base and the smoke root. | Set anywhere else: `HARNESS_SMOKE_HOLD_REFUSED <reason>` and exit 2, before any profile path is set. Plane not attached: `HARNESS_SMOKE_HOLD_UNAVAILABLE`, and the run reports its verdict as an ordinary smoke. | `walkControl.mjs` module doc, `resolveSmokeHold` | `walkControl.spec` hold arm |
| Walk manifest `<smokeRoot>/walk/manifest.json` | the held run | Written once the run holds: `planeId`, `smokeRoot`, `registryPath`, `runtimeRoot`, `identities`, `planePort`, `ingressPort`, `pid`, `candidate`, `auth`. Mode 0600, written through a rename; earlier walk files are cleared first. | No manifest: every command refuses, "no held smoke run under …". | `writeWalkManifest` | manifest and stale-request arms |
| Accepted target (`assertWalkTarget`) | AC-2 (Clio 2), RA-1 | Plane id `neo-harness-smoke`, this run's smoke root, and a registry inside that root once links resolve. Token writes go to the resolved file the check accepted. | Another plane id, root, or a registry outside the root: refused, naming the mismatch; nothing is written. | `assertWalkTarget` | refusal arm, link arm (escaping link refused, in-root link followed) |
| CLI `walkControl.mjs [--packaged] plane stop\|start \| token revoke \| token remap <identity> \| cleanup` | AC-2, AC-4 | `plane`: a control file the held run watches, answered in an ack file with `ok` and the handler's result. `token revoke`: the next registry generation holds no rows, so the plane refuses the stored token (`stale-generation`). `token remap`: the next generation binds the token to another identity the plane's graph holds. | Unknown command: usage, exit 1. No answer within 60 s: "is its window still open?". A remap to an identity the graph lacks writes nothing. | module doc, `runWalkControl` | plane and token arms on real seat-token registries |
| Plane stop answer | RA-3 | `ok: true` only once the plane's process group is gone (a forced stop included); the stop report rides the answer. | The group outlived the kill: `ok: false`, the report in the error. | `createPlaneProcess` | stop-result arms (`fixturePlane.spec`, `walkControl.spec`) |
| Control lifetime | AC-5, RA-2 | From the manifest write until teardown. `teardownBrain` closes the control first: no request starts after it, and the one in flight settles, so a plane it starts is in the drain. | A request after close is never answered. | `watchPlaneControl`, `teardownBrain` | close-race arm |
| `cleanup` | AC-7 (Emmy 3) | Removes the smoke root only once the held run's pid is gone and neither plane port listens. | While either holds: refuses, removing nothing. | `cleanupHeldRun` | cleanup arm; the installed receipt is Post-Merge Validation, read on #516 |

## Acceptance Criteria

- [ ] AC-1 (Clio 1): `NEO_HARNESS_SMOKE_HOLD=1` holds the window only in the fixture-plane arm. With no fixture plane, which also covers the operator's instance with no plane declared, the harness refuses with a named reason and never attaches (unit).
- [ ] AC-2 (Clio 2): the control script addresses only the smoke root's fixture plane, by its root, registry path and plane id. Any other root or plane id is refused, so the live plane is unreachable from it (unit). Sharpened from "compose project/port": the fixture plane is a child process, not a compose project.
- [ ] AC-3 (Clio 3): without the flag, smoke posts its verdict and exits exactly as today (the existing smoke specs stay green).
- [ ] AC-4: `token revoke` refuses the live seat token at the fixture plane and `token remap` names another account, without a restart. `plane stop` and `plane start` take the plane down and back (unit on real temp registries, plus one held run's receipt on #516).
- [ ] AC-5 (Emmy 1): the held run reuses smoke's containment and its own lifecycle: the same isolation root, the same `exitTerminal` teardown. There is no second runtime, and the six provocations are not split into features.
- [ ] AC-6 (Emmy 2): the held run names its candidate (the build info it runs) and its auth mode (the fixture seat token) when it starts. The walker drives the real UI's recovery against fixture-only faults.
- [ ] AC-7 (Emmy 3): a held run's receipt proves cleanup. The smoke root is removed and the fixture plane is stopped, while the operator's app profile and the live plane are unchanged.

## Post-Merge Validation

*(Added 2026-10-04 by the claimer. The held-run receipts in AC-4 and AC-7 need an installed candidate, so they leave the PR's gate and stay owned here, read on #516's walk.)*

- [ ] AC-4, the held run: on the installed candidate held with `NEO_HARNESS_SMOKE_HOLD=1`, `token revoke`, `token remap` and `plane stop` / `plane start` each produce the refusal or recovery the cockpit should show. One receipt each, on #516.
- [ ] AC-7, the held run: after the walker closes the window, `walkControl.mjs [--packaged] cleanup` removes the smoke root with both plane ports closed, and the operator's app profile and live plane are untouched.

## Out of Scope

- The walk itself and its receipts (#516).
- Any change to the product's boot, its plane modes, or the operator's instance.
- Faking a failure on a non-isolated instance.
- **Residual, kept (Emmy):** the fixture's seat-token mode does not certify forge-PAT expiry. A walk that needs a real expired forge PAT stays out of this leaf.

## Related

Parent: #424 (row 5). Serves: #516 (the walk), #477 (row 2's provoked states), neomjs/neo-agent-brain#571 (enrollment's recovery). Precedent: #214 and PR #350 (the fixture plane).

Sweeps:
- Live latest-open: 20 Institution issues at 2026-10-03T20:15:58Z, plus a search for "smoke hold open walker"; no equivalent.
- A2A: 30 messages in all read states; only Clio's acceptance on this scope.
- Memory Core: Clio's acceptance with its three ACs, no earlier decision.
- Own assignments: #521, #516, #512, #503, #424, #522; none overlaps.
- Structure: Institution `harness/`, beside `fixturePlane.mjs`; the Brain structure map does not apply.

Decision Record impact: `none` (test tooling inside the harness; no product contract changes).

Origin Session ID: 84371353-afea-4f59-9b58-2b8777325f56
Retrieval Hint: "smoke hold open walker isolated organism fixture plane control script revoke remap token NEO_HARNESS_SMOKE_HOLD"




## Timeline

- 2026-10-03T20:16:40Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-03T20:16:41Z @neo-opus-ada added the `enhancement` label
- 2026-10-03T20:16:41Z @neo-opus-ada added the `agent-os` label
- 2026-10-03T20:16:41Z @neo-opus-ada added the `ai` label
- 2026-10-03T20:16:41Z @neo-opus-ada added the `testing` label
- 2026-10-03T20:16:48Z @neo-opus-ada added parent issue #424
- 2026-10-03T20:16:50Z @neo-opus-ada marked this issue as blocking #516
- 2026-10-03T20:16:51Z @neo-opus-ada cross-referenced by #424
- 2026-10-04T10:01:59Z @neo-opus-ada added this to the **FM v1** milestone
- 2026-10-04T10:59:50Z @neo-fable cross-referenced by #351
- 2026-10-04T11:01:22Z @neo-opus-ada cross-referenced by #533
- 2026-10-04T11:10:05Z @neo-fable cross-referenced by #534
- 2026-10-04T11:10:14Z @neo-fable marked this issue as blocking #534
- 2026-10-04T11:44:36Z @neo-opus-ada cross-referenced by PR #537
- 2026-10-04T11:56:03Z @neo-gpt-emmy cross-referenced by #12
- 2026-10-04T12:22:42Z @neo-opus-ada referenced in commit `ecb0626` - "fix(harness): the walk control writes only inside its root, closes before the drain, and answers a stop only once the plane is gone (#523)

Three boundary defects from the #537 review, each reproduced on 130a7f5d:

- a registry reached through a link inside the smoke root was written outside it: assertWalkTarget compares resolved paths and hands the write the file it accepted;
- a plane start queued while the window closed could register a child after teardown took its snapshot: the watcher's close() stops admission and settles the request in flight, and teardownBrain awaits it first;
- a stop whose process group outlived the kill was acknowledged ok: the plane's stop throws unless its group is gone, and the answer carries the stop report.

Each new arm fails on the previous source."
- 2026-10-04T12:38:50Z @tobiu referenced in commit `606ab2a` - "feat(harness): a walker can hold the fixture-plane smoke's organism open and drive its plane and seat token (#523) (#537)

* feat(harness): a walker can hold the fixture-plane smoke's organism open and drive its plane and seat token (#523)

NEO_HARNESS_SMOKE_HOLD=1 acts only in the fixture-plane arm; set anywhere else the harness refuses before anything starts. A held run stops after the plane observations with the organism attached and the window open, names its candidate and auth mode, and writes a walk manifest under its smoke root. Closing the window quits through window-all-closed and will-quit, the owned-Brain teardown exitTerminal runs, so the organism and the plane stop with it; a held run arms no timeout net.

harness/walkControl.mjs acts only on what that manifest names (the fixture plane id, the smoke root, a registry inside it): plane stop and start go through a control file the held run watches and acknowledges, token revoke and remap rewrite the fixture's seat-token registry, which the plane's verifier reads on its next request, and cleanup removes the smoke root once the run and both plane ports are gone.

The fixture plane's child is now a createPlaneProcess handle registered as drain-owned but never a Brain claim: it stands in for a remote plane, whose outage the shell meets through its own reads, never as a crash of a child it owns. A held run seeds a second identity, so a remap names another account the plane holds. walkControl.mjs joins the packaged main's module set.

* fix(harness): the walk control writes only inside its root, closes before the drain, and answers a stop only once the plane is gone (#523)

Three boundary defects from the #537 review, each reproduced on 130a7f5d:

- a registry reached through a link inside the smoke root was written outside it: assertWalkTarget compares resolved paths and hands the write the file it accepted;
- a plane start queued while the window closed could register a child after teardown took its snapshot: the watcher's close() stops admission and settles the request in flight, and teardownBrain awaits it first;
- a stop whose process group outlived the kill was acknowledged ok: the plane's stop throws unless its group is gone, and the answer carries the stop report.

Each new arm fails on the previous source."
- 2026-10-04T12:38:50Z @tobiu closed this issue

