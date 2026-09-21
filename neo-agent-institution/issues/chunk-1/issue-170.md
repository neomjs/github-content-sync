---
id: 170
title: 'An instance switch throws: the cockpit has no reconnectFleet()'
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-09-19T10:32:04Z'
updatedAt: '2026-09-19T13:49:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/170'
author: neo-fable-clio
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
closedAt: '2026-09-19T13:49:21Z'
---
# An instance switch throws: the cockpit has no reconnectFleet()

## Context

Found by @neo-opus-grace while reviewing PR #162 (2026-09-18, executed, not grepped: `typeof FleetCockpit.prototype.reconnectFleet` is `undefined`). Re-verified at `dev@d02fe83` on 2026-09-19. It is the second instance of #161's class: since the #50 rebuild a method that callers OUTSIDE the cockpit use lives only on the controller chain, and the Container kept no public delegate.

## The Problem

`ViewportController` calls `reconnectFleet()` on the cockpit **component** in two flows:

- `switchToProfile()` (`apps/agentos/view/ViewportController.mjs:195`), after custody is established and `instanceState: 'starting'` is written.
- `onConnectPlane()`'s success branch (`:395`), after the `tenant connected` notice is written.

`reconnectFleet()` exists only on `LivenessController` (`apps/agentos/view/fleet/cockpit/LivenessController.mjs:802`), which the cockpit's `Controller` extends. The optional chain guards a missing cockpit, not a missing method, so with a mounted cockpit both calls throw `TypeError: cockpit.reconnectFleet is not a function`.

Consequences, read from the code and not yet witnessed in the running app:

- `switchToProfile()` rejects before `await verified`: the transition verdict is never settled, `instanceState` stays `'starting'`, and the cockpit is never re-driven against the chosen instance. `onSwitchInstance()` neither awaits nor catches, so the rejection is unhandled.
- `onConnectPlane()` rejects after a success notice: the re-drive that branch asks for never runs.

No unit arm runs either call site against the real `FleetCockpit.prototype`. `custodyHeal.spec.mjs` sets its own `reconnectFleet` on a controller, which is why the suite is green.

## The Architectural Reality

- `AgentOS.view.fleet.cockpit.Container` already exposes `loadRoster()` as a documented public delegate to `getController()` (`cockpit/Container.mjs:714`, from #161), with the contract "a parent never reaches into this view's controller".
- Inside the cockpit the Reconnect button reaches the method through the controller chain (`handler: 'reconnectFleet'`, `cockpit/Container.mjs:350`). The inside path works; the outside path does not.
- `cockpit/Container.mjs` is 992 lines. `buildScripts/checkAppFileSizes.mjs` errors above 1,000.

## The Fix

`Container#reconnectFleet()`: a public delegate beside `loadRoster()`, same shape and doc contract. Both `ViewportController` call sites stay as written.

Red-first unit arms drive `switchToProfile()` and `onConnectPlane()`'s success branch with a cockpit built FROM `FleetCockpit.prototype`, only `getController` stubbed (PR #162's arm is the pattern).

## Contract Ledger

*(Backfilled 2026-09-19 on @neo-gpt-emmy's review of PR #173: the public surface was documented in prose only.)*

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `AgentOS.view.fleet.cockpit.Container#reconnectFleet()` — a new public method beside `loadRoster()` in `apps/agentos/view/fleet/cockpit/Container.mjs` | #161's boundary, stated on the `loadRoster()` delegate: a parent never reaches into this view's controller. Consumers: `ViewportController#switchToProfile()` and `#onConnectPlane()` | Delegates to the cockpit controller's `reconnectFleet()` (`LivenessController`). Returns nothing: the re-drive is fire-and-forget, and both consumers ignore a result | A missing cockpit stays the consumers' concern (their optional chain skips the call). The delegate adds no guard of its own, matching `loadRoster()`; no consumer falls back to `getController()` | JSDoc `@summary` on the method, the same shape as `loadRoster()` | `unit/apps/agentos/ViewportController.spec.mjs`: the two call-site arms over a cockpit built from `FleetCockpit.prototype` (red on `dev` with the TypeError), plus the sweep arm that pins every method the Viewport calls on the cockpit |

## Acceptance Criteria

- [ ] `FleetCockpit.prototype.reconnectFleet` exists, delegates to the controller, and is documented like `loadRoster()`.
- [ ] A unit arm drives `ViewportController#switchToProfile()` against a prototype-built cockpit: red on `dev` with the TypeError, green on the branch. It asserts the re-drive ran and that `instanceState` settles (`'off'` on a failed verification), never staying `'starting'`.
- [ ] The same for `onConnectPlane()`'s success branch.
- [ ] Class sweep, listed in the PR body: every method `Viewport` and `ViewportController` call on the `fleet-cockpit` reference exists on `FleetCockpit.prototype`.
- [ ] `buildScripts/checkAppFileSizes.mjs` reports no error for `cockpit/Container.mjs`.

## Out of Scope

- The cockpit's seam cut (three files in the size warn band): #42, #24.
- Any change to `LivenessController#reconnectFleet()`.

## Avoided Traps

- `cockpit.getController().reconnectFleet()` from `ViewportController`: rejected by #161's contract, a parent never reaches into another view's controller.
- A hand-made cockpit stub that carries its own `reconnectFleet`: that is how the defect stayed green. Stubs come from the real prototype.

## Related

#161 and PR #162 (first instance, `loadRoster`) · #50 (the rebuild)

Decision Record impact: none

unowned-rationale: parked claimable as a first Fleet Manager leaf for a waking peer. It does not move the visual stamp and does not touch the open card stack (PR #169, #168). The filer takes it after that stack if it is still open.

Live latest-open sweep: all 16 open issues read at 2026-09-19T10:31:33Z, no equivalent. A2A claim sweep (latest 12, all read-states, back to 2026-09-18T17:49Z): no claim on this scope. Memory Core sweep keyed on "reconnectFleet undefined on the cockpit, switchToProfile throws": no prior decision. Own open assignments (#10, #123, #168): none on this surface.

Origin Session ID: 49c5e4f0-ac44-436d-898f-3e3fb0c349dc
Retrieval Hint: "reconnectFleet undefined FleetCockpit prototype ViewportController switchToProfile"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 49c5e4f0-ac44-436d-898f-3e3fb0c349dc

## Timeline

- 2026-09-19T10:32:05Z @neo-fable-clio added the `bug` label
- 2026-09-19T10:32:05Z @neo-fable-clio added the `agent-os` label
- 2026-09-19T10:32:05Z @neo-fable-clio added the `ai` label
- 2026-09-19T11:00:05Z @neo-fable-clio cross-referenced by #171
- 2026-09-19T13:05:45Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-19T13:10:56Z @neo-fable-clio cross-referenced by PR #173
- 2026-09-19T13:32:48Z @neo-fable-clio referenced in commit `76ae3c9` - "fix(cockpit): the cockpit answers reconnectFleet() — an instance switch settles instead of throwing (#170)

ViewportController calls reconnectFleet() on the cockpit component after an instance switch and after a tenant connects, and the method lived only on the controller chain: both flows threw a TypeError, a switch stayed in 'starting', and a connected tenant never got its re-drive. The cockpit gains the public delegate beside loadRoster(). Unit arms drive both call sites against a cockpit built from the real class, and a sweep arm pins every method the Viewport calls on it. A dead WorkspaceDocument import leaves the file, which keeps it under the size bar."
- 2026-09-19T13:49:21Z @tobiu referenced in commit `6d953f1` - "Merge pull request #173 from neomjs/agent/170-cockpit-reconnect-delegate

fix(cockpit): the cockpit answers reconnectFleet() — an instance switch settles instead of throwing (#170)"
- 2026-09-19T13:49:21Z @tobiu closed this issue

