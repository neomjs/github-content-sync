---
id: 849
title: A boot refusal that is not the credential's needs a new PAT to recover
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-31T00:05:12Z'
updatedAt: '2026-10-31T15:01:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/849'
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
closedAt: '2026-10-31T15:01:58Z'
---
# A boot refusal that is not the credential's needs a new PAT to recover

## Context

Operator, 2026-10-30 (prio 0, the incident in #847): after the FM's login wait ended on "not a Neo plane", the operator could not get back without generating a new PAT. The stored PAT was valid: nothing had revoked it, and the plane admits PATs the same way minutes later. #847 stops the wait from ending while the plane starts. This leaf is the way back from the refusals that can still end a boot.

## The Problem

Every boot refusal's lead points at Connect:

- "Connect to the plane's own address."
- "Bring that plane back, or connect to another."
- "The plane refused this shell — connect it again"

(`SpineBanner` `PLANE_REFUSALS` and the `plane-refused` verdict, each with the `connect-plane` action.) Connect always prompts for a PAT (`createPlaneBroker().attach` → `promptCredential`), and the shell never retries its stored record once a boot has settled.

So any refusal that is not about the credential costs a PAT the operator may not have at hand. Since #848 a starting plane's 5xx no longer ends a stored plane's boot. The cases left are a Fleet child that exits while the plane still accepts the stored PAT (`plane-refused`) and an answer at the address that is not a plane (`plane-not-a-plane`, e.g. a 4xx while the plane starts). Rotating the PAT also leaves the old one live.

## The Architectural Reality

- `harness/planeConfig.mjs` `createPlaneBroker` offers `status`, `attach` (prompt → probe → write → relaunch) and `verify`. Nothing retries with the stored record.
- `harness/main.mjs` runs `bootProductBrain()` once. Its `.catch` settles the boot down (`appLifecycle.settleBrainBoot(false, cause)`). `fleetCapability` reads `brainBootPromise` at call time, so a later Fleet read joins whichever boot main retains.
- `harness/appLifecycle.mjs` keeps the first cause of a degraded episode (first-cause-wins within a severity tier). A second boot in the same episode would keep the first boot's refusal.
- `apps/agentos/util/SpineBanner.mjs` gives every boot refusal the `connect-plane` action. The cockpit bar has one action button (`ControlToolbar` `fleet-reconnect-button`). The setup card's Connect door is also reachable from the shell switcher and Home's doors (`ViewportController.showPlaneSetup`).
- ADR 0034 §2.3 item 8, in this repository since #686, names the plane broker's capabilities one by one, and `harness/preload.cjs` says additions amend §2.3 first.
- #843 (open) covers the other half of the cost: Connect's relaunch. This leaf must not add a relaunch.

Design authority: #425 Avoided Traps — "Re-probing on a timer. One probe per failed boot is enough. Recovery is the operator's Connect." That ruling assumed Connect costs nothing. It costs a PAT. Retry stays the operator's click, never a timer.

## The Fix

1. ADR 0034 §2.3 item 8 gains a fourth capability, `retryPlane()` → `shell-plane-retry`. Main re-runs the boot with the record it launched with: no prompt, no relaunch, and only after a boot settled down with a retryable refusal. The reply is `{ok, reason}` and never carries the PAT. The amendment lands in this leaf's PR.
2. Retryable: `plane-refused`, `plane-not-a-plane` and `plane-unreachable`, one exported set in `harness/brain.mjs` beside `bootFailureCause`. The banner mirrors it, and a spec asserts the two agree. The credential's two refusals and `organism-beside-plane` keep Connect: their fix is a PAT or another plane, and a retry would meet the same answer.
3. Main refuses a retry while a boot runs, after one came up, and while quitting. An accepted retry opens a new boot episode in `appLifecycle` (the old refusal stops being the retained cause; an owned-child fault is never erased), restarts the transport fact at `starting`, and replaces `brainBootPromise`.
4. The banner's action reads Retry for the retryable refusals, and each lead names both ways out. The click calls `retryPlane()`, then re-drives the cockpit's liveness, whose reads join the retried boot.
5. Connect stays on the setup card, for a different plane or a different PAT.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| `neoShell.retryPlane()` → `shell-plane-retry` (`harness/preload.cjs`, `createPlaneBroker().retry`) | ADR 0034 §2.3 items 4 and 8 | A trusted sender on a packaged shell reaches main's boot retry; the reply is `{ok, reason}` | An untrusted sender throws, as for its siblings; unpackaged → `not-packaged`; main refuses with `boot-running`, `boot-up`, `not-retryable` or `quitting` | ADR 0034 §2.3 item 8 | `planeConfig.spec.mjs`; main's smoke key census; `preload.spec.mjs` |
| `RETRYABLE_BOOT_REFUSALS` and the retry gate (`harness/brain.mjs`) | This ticket | Accepts a settled boot with `up: false` whose `cause.source` is in the set | A boot without a typed cause (`boot-not-ready`) is not retried | JSDoc | `brain.spec.mjs` |
| The boot episode (`harness/appLifecycle.mjs`) | Its first-cause-wins rule | A retry clears the retained refusal; the state stays `degraded` until the retried boot settles and records its own cause | Refused after an owned-child fault, whose cause stays | JSDoc | `appLifecycle.spec.mjs` |
| `SpineBanner` verdict `action` | #477 (a reason and a next step) | `retry-plane` for the retryable causes; the bar's button reads Retry | `connect-plane` unchanged for the credential's two and `organism-beside-plane` | `learn/CockpitStateCensus.md` | `spineBanner.spec.mjs` |
| `ShellPlane.retryPlane()` (`src/main/addon/ShellPlane.mjs`) | Its sibling `probePlane` | Relays `neoShell.retryPlane()` | No shell → `{ok: false, reason: 'no-shell'}` | JSDoc | — |

## Acceptance Criteria

- [ ] AC-1: a boot that settled with a retryable refusal recovers through Retry, with no PAT prompt and no relaunch. Unit test with broker and boot seams.
- [ ] AC-2: the credential's own two refusals and `organism-beside-plane` offer Connect only. Spine banner spec.
- [ ] AC-3: Retry never carries the PAT to the renderer. Spec.
- [ ] AC-4: a retried boot that fails again shows its own refusal, not the first one. `appLifecycle` spec.
- [ ] AC-5: ADR 0034 §2.3 item 8 names `retryPlane()`, and main's preload key census and `preload.spec.mjs` list it.
- [ ] AC-6 `[L4-deferred]`: on the installed FM, a boot refusal followed by Retry reaches `live`. Receipt on #516.

## Out of Scope

- A cold Memory Core that cannot reach GitHub answering a valid PAT with 401 (#847, Out of Scope).
- Adopting a new PAT without a relaunch (#843).
- A boot that failed without a typed refusal (`boot-not-ready`).

## Related

Part of #424 · #847 · #843 · #425 · promotes the defect-note "the FM cockpit's boot-refusal recovery is wrong …" (2026-10-30T23:49Z) on the operator's escalation.

Live latest-open + closed sweep: the latest 20 open issues and closed searches ran at 2026-10-30T23:47Z for #847 and were re-checked at the newest open issues just before filing. #425 and #446 were read (#446: a PAT refused mid-session gets Connect). No equivalent. A2A sweep: no claim on this scope. Own-assignment sweep: #843 is adjacent (new-PAT relaunch), not equivalent. Structure map: N/A (existing `harness/` and `apps/agentos/`). Decision Record impact: amends ADR 0034 §2.3 item 8 (one more broker capability), in this leaf's PR.

Intake 2026-10-31: #848 merged after filing and touched `SpineBanner.mjs`, `main.mjs` and `planeConfig.mjs`. The premise holds (The Problem); the Fix, the ledger and AC-2, AC-4 and AC-5 come from that read. Sweep re-run 2026-10-31T14:0xZ: no newer equivalent.

Retrieval Hint: "FM boot refusal retry stored PAT Connect demands new PAT rotated"


## Timeline

- 2026-10-31T00:05:13Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-31T00:05:14Z @neo-opus-ada added the `enhancement` label
- 2026-10-31T00:05:14Z @neo-opus-ada added the `agent-os` label
- 2026-10-31T00:05:14Z @neo-opus-ada added the `ai` label
- 2026-10-31T00:05:15Z @neo-opus-ada added parent issue #424
- 2026-10-31T14:28:23Z @neo-opus-ada cross-referenced by PR #850
- 2026-10-31T14:37:54Z @neo-opus-ada referenced in commit `5f0f066` - "chore(agentos): the banner action leaves the 999-line cockpit controller for SpineAction (#849)

Controller.mjs sat one line under the 1,000-line app bar, so Retry's dispatch failed checkAppFileSizes. The banner's one action (Connect, Retry, Reconnect) moves to apps/agentos/util/SpineAction.mjs beside PlaneCredentialCheck; the controller keeps a one-line handler and ends at 993 lines. A spec comment loses its ticket id (check-ticket-archaeology)."
- 2026-10-31T15:01:58Z @tobiu referenced in commit `8cf468c` - "feat(agentos): Retry runs a refused boot again with the stored PAT, no relaunch (#849) (#850)

* feat(agentos): Retry runs a refused boot again with the stored PAT, no relaunch (#849)

A boot the plane refused without a word against the stored PAT (plane-refused,
plane-not-a-plane, plane-unreachable) offered only Connect, which always asks for a new PAT.
The banner's action is now Retry for those three: main runs the boot again in process with
the record it launched with, with no prompt and no relaunch. ADR 0034 §2.3 item 8 gains
retryPlane(). The lifecycle opens a new boot episode, so a retried boot that fails again shows
its own refusal instead of the first one. The credential's two refusals and
organism-beside-plane keep Connect.

Two banner goldens move (not-a-plane, plane-unreachable: Retry and its lead). The stamp keeps
its engine line at 95e63df6 by hand, as in #848: --stamp writes the running engine.

* chore(agentos): the banner action leaves the 999-line cockpit controller for SpineAction (#849)

Controller.mjs sat one line under the 1,000-line app bar, so Retry's dispatch failed checkAppFileSizes. The banner's one action (Connect, Retry, Reconnect) moves to apps/agentos/util/SpineAction.mjs beside PlaneCredentialCheck; the controller keeps a one-line handler and ends at 993 lines. A spec comment loses its ticket id (check-ticket-archaeology)."
- 2026-10-31T15:01:59Z @tobiu closed this issue
- 2026-10-31T15:10:37Z @neo-opus-ada cross-referenced by #516

