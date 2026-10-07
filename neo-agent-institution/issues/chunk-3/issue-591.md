---
id: 591
title: Fleet pop-out windows cannot find a drop target on return
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-07T12:50:27Z'
updatedAt: '2026-10-07T15:26:00Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/591'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 12
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-07T15:22:16Z'
---
# Fleet pop-out windows cannot find a drop target on return

## Context
The operator and Sophie reported that a pane can be torn out of the installed Fleet Manager, but moving the resulting window back over the cockpit shows no drop zones. Grace traced the missing cross-window registration and routed the consumer repair here; neomjs/neo#19448 separately owns the Engine warning.

The source gap is verified at Institution `3b68995f1a5e9b0011feae3d2301dddd7abfa329`: the cockpit enables the tear-out lifecycle, but its inherited `VesselContainer.getDockProjectionOptions()` supplies only the base tear-out bundle and in-window drag callbacks. It publishes no cross-window sort group. This is a source explanation of the reported symptom, not an installed repair witness.

## The Problem
Tear-out and cross-window participation are separate Engine opt-ins. Without a sort group, the adapter gives tab zones no coordinator group, and `ParticipationLifecycle.syncParticipation` retires/omits the workspace's target. The existing vessel-death return test proves automatic return after closing a popup, not moving the popup back over the cockpit.

The runtime boundary check also found why group publication alone is insufficient. Fleet's intentional `resolvePane()` placeholder is freshly allocated on each query while a pane is vesseled. Running the exact resolver methods twice produced different `draggedItem` objects, which the coordinator's native-drop identity fence rejects before parking. Fleet also supplies no native park/resume/retire callbacks or proxy embodiment; the strict coordinator requires those before committing.

## The Architectural Reality
Design authority: `Neo.dashboard.dock.Workspace` documents that a workspace publishing `crossWindowSortGroup` composes its cross-window Participation, independently of the tear-out flag. Fleet's `VesselContainer` class owns the platform pop-out/tear-out/return seams; `Container` inherits that shared layer.

Use the Workstation's named static sort-group pattern. This is discovery identity, not commit authority: `DragCoordinator.admitsOwnership` and `resolveClaimedTarget` exclude a foreign topology Group before target hit-testing, and Participation rechecks ownership on commit. Do not create a second group-ownership mechanism.

The Engine projection already passes the workspace component as `dockWorkspaceBoundaryContainerId`; the adapter falls back to it when no explicit tear-out boundary is supplied. Add a separate boundary only if the actual consumer witness requires a different root.

## The Fix
Publish Fleet's shared cross-window sort group from the existing cockpit `VesselContainer.getDockProjectionOptions()`, preserving the inherited lifecycle handlers and in-window callbacks. Verify the actual popup/native return path and reuse the Engine participation/transfer primitives rather than hand-moving panes.

Supply the existing `getDockParticipationConfig()` native-drag seams from the vessel layer, resolving the stable captured `vesselPane()` instead of changing the projection's placeholder policy. Compose the existing Engine proxy embodiment and VesselPark owners, then recompose Participation after those collaborators exist; retire them with the view.

This repair is native popup return to the main cockpit only. Follow Workstation's native-main policy: the real popup remains physically in place during the bounded handoff, rejection restores its pane, and only a committed drop retires the exact Group-owned window. Do not add popup-to-popup geometry, focus/move machinery or a copied transfer algorithm. Unknown source/target/ownership refuses.

Keep regression coverage in the existing cockpit projection/vessel or tear-out tests. No new runtime module or Engine pin is required.

## Contract Ledger
| Surface | Authority | Behavior | Fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| Fleet projection options | Workspace cross-window opt-in; Workstation precedent | Projected Fleet zones publish one shared coordinator group; inherited tear-out and drag handlers remain | Existing Engine refusal for unresolved or foreign ownership | Existing method JSDoc | Red/green consumer registration/projection control |
| Native drag source | Existing Participation callback; Fleet held-pane capability | Repeat reads return the same captured live pane with current workspace/Group identity, never a projection placeholder | Missing/wrong owner, window or pane refuses | New callback JSDoc | Stable-identity and negative controls |
| Native handoff lifecycle | Engine VesselPark/proxy embodiment; Workstation main-target precedent | Restore on rejection; retire exact owned window only after semantic commit and preserve retry on refused close | No physical move/focus on main-target handoff | Composition and lifecycle JSDoc | Controlled park/restore/disposition tests |
| Torn-out window return | Engine Participation and topology Group | Returning the window discovers the cockpit target and restores the same pane through the existing commit path | Refused targets do not transfer ownership | Existing vessel lifecycle contract | Isolated return witness; installed follow-through below |

## Acceptance Criteria
- [ ] A regression control fails on the current consumer and passes with the change: its projected zones/Participation register the expected cross-window target.
- [ ] The isolated return witness moves a committed vessel back to the cockpit through the real Engine return path, with the same pane instance and no duplicate ownership; preserve the distinction between service-level and native-frame evidence.
- [ ] Repeated native-source queries retain the same live pane identity; wrong-window, missing ownership and missing live-pane cases refuse while projection placeholders remain unchanged.
- [ ] The composed native lifecycle restores on rejection, retires only on accepted commit, and retains retry authority when close is refused.
- [ ] Existing in-window drag and vessel-death return coverage remains green.

## Post-Merge Validation
- [ ] [L4-deferred — operator handoff needed] On the next installed candidate, tear out a pane, drop outside, move its native window back over the cockpit, observe drop zones and return the same live pane. Residual-Owner: #12.

## Decision Record impact
Aligned with the existing DockLayouts cross-window ownership/participation contract. No new Engine default or ownership policy.

## Out of Scope
Popup sizing, the returned-header positioning defect, admission/credentials, and the Engine warning in neomjs/neo#19448.

## Avoided Traps
A per-workspace sort group would prevent discovery. Deriving a second group identity is unnecessary because the coordinator already gates ownership before candidates. Enabling tear-out alone or verifying only return-on-close does not prove the requested gesture.

## Related
#12; #382 (in-window feedback); #455 (prior Engine pin/adoption); neomjs/neo#19448.

Live latest-open sweep: latest 20 open Institution issues and 30 all-state A2A messages checked 2026-10-07 12:48Z; no equivalent consumer leaf. No open Institution PR.
MC sweep: symptom query recovered Grace's current source diagnosis and the earlier in-window feedback repair; these have different close targets.
Own-assignment sweep: #590 is launch-admission feedback; #42 is a view-layer debt investigation, neither covers this return defect.
Structure map: N/A — existing Institution view methods and their existing tests; no Brain placement change.

Origin Session ID: 7cdef292-c073-447b-9afd-4eaab22ecdbf

Retrieval Hint: "Fleet native window return no drop zones crossWindowSortGroup VesselContainer"




## Timeline

- 2026-10-07T12:50:27Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-07T12:50:29Z @neo-gpt-emmy added the `bug` label
- 2026-10-07T12:50:29Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-07T12:50:30Z @neo-gpt-emmy added the `ai` label
- 2026-10-07T12:51:13Z @neo-gpt-emmy added parent issue #12
- 2026-10-07T13:56:07Z @neo-opus-grace cross-referenced by PR #19449
- 2026-10-07T14:50:51Z @neo-gpt-emmy referenced in commit `f23ce84` - "test(dock): refresh Fleet return visual receipt (#591)"
- 2026-10-07T14:56:51Z @neo-gpt-emmy cross-referenced by PR #595
- 2026-10-07T15:22:05Z @tobiu referenced in commit `46929be` - "feat(dock): wire Fleet native window return (#591) (#595)

* feat(dock): wire Fleet native window return (#591)

* test(dock): refresh Fleet return visual receipt (#591)"
- 2026-10-07T15:22:17Z @tobiu closed this issue

