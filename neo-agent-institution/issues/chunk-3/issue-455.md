---
id: 455
title: Carry the Engine mount-edge fix with Fleet pop-out compatibility
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - build
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-02T17:15:47Z'
updatedAt: '2026-10-02T17:49:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/455'
author: neo-gpt-emmy
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
blocking:
  - '[x] 454 The memories pane''s edge replay retires once the engine pin re-arms a mounted body'
closedAt: '2026-10-02T17:49:10Z'
---
# Carry the Engine mount-edge fix with Fleet pop-out compatibility

## Context

The next Engine pin must carry the merged grid mount re-arm from neomjs/neo#19364 so #454 can retire Fleet's memory-edge replay. Current Institution dev `99e2acf` pins Engine `9376944`. The frozen target is `82bc6158444306e0c342e8cda480e77158c9fedb`, which also carries neomjs/neo#19358, neomjs/neo#19360, neomjs/neo#19363 and neomjs/neo#19365. Brain remains `447d96e`.

## The Problem

A pin alone breaks the existing click pop-out. `VesselContainer.popOutPane:483` calls `handleDockPopOutAction`, now owned by the Engine's HeaderActions plugin rather than Workspace. Its `measureDockPaneRect` override is also bypassed: the new plugin measures itself. An exact-method control with an injected host measured the old override called once and passing the inspector's 480×640 composition; the new handler called it zero times and admitted the generic measured rectangle instead. No live window was opened for that control.

## The Architectural Reality

The Engine retains two owner seams: `Workspace.onDockHeaderAction` dispatches an action to its configured plugin and returns the result (or null if unanswered); `Workspace.admitDockPopOut` is explicitly the host-owned admission hook called by the plugin. Fleet's designed composition belongs at that admission boundary. The Engine owns measurement and detach/commit behavior; the product must not copy those handlers or revive them on its own class.

The remaining Engine source delta adds `scrollEdge.total` as information and clears its visible-count latch on mount. The current Fleet consumers do not read the new informational total, so #19360 requires no additional adapter. #19363 is the ADR-only declaration of the already-merged `verifyPlane` consumer; #19365 adds tests only. Neither changes this product's runtime API. Engine package dependency declarations do not change. #454's separate replay retirement stays with Vega; the existing in-flight gate makes pin-first ordering valid.

## The Fix

1. Advance the Engine manifest/lock pin to the frozen target, retaining Brain447d96e.
2. Route `popOutPane` through `onDockHeaderAction({action:'pop-out', dockNodeId, tabContainer})`; an unanswered action reports a refusal without document mutation.
3. Replace the obsolete geometry override with `admitDockPopOut`: pass the designed composition for a named pane, otherwise retain the measured proxy rectangle, then invoke the inherited admission hook. Do not mutate the caller's descriptor.
4. Keep the existing real-object vessel tests; update their geometry entrypoint and add coverage for an unanswered/declined action. Refresh the visual input stamp only after the unchanged goldens pass.

Prescription checked: the two surviving Workspace owner seams own dispatch and host admission; no Engine facade expansion or product copy of plugin behavior is needed.

## Contract Ledger

| Surface | Authority | Behavior | Edge | Docs | Evidence |
|---|---|---|---|---|---|
| Engine dependency | merged target SHA | manifest/lock select82bc6158 | Brain447d96e unchanged | PR census | locked install + reference checks |
| Fleet popOutPane | Workspace.onDockHeaderAction | uses configured engine action, preserves result envelope | null/unanswered refuses, no detach | method/class JSDoc | vessel happy/refused/declined arms |
| Fleet admission geometry | Workspace.admitDockPopOut and vesselCompositions | inspector keeps480×640; other panes retain measured rect | descriptor unchanged, inherited refusal propagates | override JSDoc | admission controls + browser pop-out |
| Visual/gesture surface | committed goldens and existing NL journeys | no intended appearance or ownership change | no silent snapshot rewrite | PR receipts | visuals + pop-out/tab-drag journeys |

## Acceptance Criteria

- [ ] Engine manifest/lock select the exact target; Brain is unchanged.
- [ ] Click pop-out reaches the public dispatch, preserves the designed inspector geometry, and keeps the existing admission/commit refusal behavior.
- [ ] An unanswered action refuses without mutation; the host geometry override preserves generic measured geometry and does not mutate its input.
- [ ] Existing unit/component checks, Fleet pop-out and tab-drag NL journeys, and the Darwin visual suite pass. The input stamp matches after validation.

## Out of Scope

#454's memory replay deletion; a Brain bump; new header behavior; package installation or live seat interruption.

Decision Record impact: aligned-with ADR0029's existing host/Engine ownership, no amendment.

Related: #454 · #12 (Engine19358's post-merge cockpit witness) · neomjs/neo#19358 · neomjs/neo#19364.

Sweeps: current latest20 open Institution issues and all-state recent A2A checked immediately before creation; no competing Engine pin. Own assignment42 is measurement, prior451 is merged/closed. Raw-memory probes did not supply a current decision; exact source and the isolation control govern. Named compatibility fork sent to the Engine extraction's author and to454's owner. No new module is prescribed.

Origin Session ID: 8d1cf4b5-75d2-4880-8358-873e0ac47fe0
Retrieval Hint: Engine82bc6158 HeaderActions VesselContainer admission geometry scrollEdge mounted pin


## Timeline

- 2026-10-02T17:15:47Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-02T17:15:48Z @neo-gpt-emmy added the `enhancement` label
- 2026-10-02T17:15:49Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-02T17:15:49Z @neo-gpt-emmy added the `ai` label
- 2026-10-02T17:15:49Z @neo-gpt-emmy added the `build` label
- 2026-10-02T17:16:40Z @neo-gpt-emmy marked this issue as blocking #454
- 2026-10-02T17:30:27Z @neo-gpt-emmy cross-referenced by PR #458
- 2026-10-02T17:31:31Z @neo-gpt-emmy cross-referenced by #12
- 2026-10-02T17:49:09Z @tobiu referenced in commit `6783360` - "feat(deps): align Fleet pop-outs with the new Engine (#455) (#458)"
- 2026-10-02T17:49:10Z @tobiu closed this issue

