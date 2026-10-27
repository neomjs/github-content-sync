---
id: 690
title: Keep Fleet header counts current after roster lifecycle changes
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - design
  - testing
assignees:
  - neo-gpt-euclid
createdAt: '2026-10-14T10:45:09Z'
updatedAt: '2026-10-14T13:32:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/690'
author: neo-gpt-euclid
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
closedAt: '2026-10-14T13:32:08Z'
---
# Keep Fleet header counts current after roster lifecycle changes

## Context

The operator reported on 2026-10-14 that roster play/stop leaves the header result unchanged: `4 started · 8 excluded` beside the correct `Stop fleet · 8`. An independent read of the installed candidate reproduces the ambiguity after another roster change: `8 started · 4 excluded` beside `Start fleet · 1`.

Binding: Institution `e5aa4238`, Engine `af539a9f`, Brain `d628999a`; Fleet route, team profile; toolbar 1946×44 CSS px. This was a read-only inspection; no peer Start/Stop was triggered.

Design authority: the operator's request that the header update after roster lifecycle actions; #477's truthful-state outcome; closed `#618` distinguishes a batch's `N started/stopped · M excluded` result from the fleet button's current partition. Active operation feedback remains distinct; the settled header now follows current roster state.

## The Problem

A historical batch result remains unlabelled in the header while the adjacent button follows the current roster. The unchanged result looks like current fleet status. A late answer can also restore an older result while its old batch token remains current.

## The Architectural Reality

`FleetBatchController.renderStartSummary/renderSummarySlot` owns the shared result/armed-stop slot. Its live `onRosterSettled` method only refreshes the stop arm and fleet button. `Controller.onAgentLifecycleIntent` dispatches individual card actions and schedules roster reconciliation without retiring that result. `onRosterRetired` fences the old batch but only clears the slot when a stop was armed. The existing start/stop partitions remain lifecycle authority; presence counts are separate.

Reproducer: render a four-start/eight-excluded batch result, admit an eight-up roster, then call `onRosterSettled`. The button becomes Stop fleet · 8 and the old result remains.

## The Fix

The operator clarified the target: Gemini's unbenching legitimately makes Start fleet · 1, and the excluded count must change from four to three. The primary settled header is current fleet state, not a frozen last-command result.

Project current started/up, ready, pending, unknown and excluded cohorts from the admitted roster using the existing participation/runtime eligibility rules and start/stop partitions. Render the current counts after roster admission and individual actions. Active batch feedback and armed-stop confirmation may temporarily own the slot, then yield back to current state. Fence late batch writers when ownership returns; preserve their per-member uncertainty/rejection evidence on the records. No mutable count authority, new Brain verb or lifecycle-admission change.

## Contract Ledger

| Surface | Authority | Behavior / fallback | Docs / evidence |
|---|---|---|---|
| Settled fleet header | Admitted roster + existing participation/runtime eligibility | Current up/ready/pending/unknown/excluded cohorts; 8 up + 3 excluded after unbench | Owner JSDoc; unit and rendered fixture |
| Live button | Existing start/stop partitions over admitted records | Continue to show the actual next plan; no invented started count | Existing planner; exact 4→8 and stop→play controls |
| Individual intent / roster retirement | Existing controller ingress and admission hooks | Retire obsolete feedback and old late writers; keep an active batch's work | Controller JSDoc; late-answer/profile controls |
| Armed stop and diagnostics | Existing stop-arm binding and adapter outcomes | Confirmation keeps the slot; UNKNOWN/refusal evidence is not silently resolved | Existing stop/wave tests and negative controls |

## Acceptance Criteria

- [ ] The settled header shows current fleet counts from the admitted roster; active operation feedback is distinct and cannot permanently replace them.
- [ ] Four-start→eight-up and fleet-start→card-stop→card-play update current started/up counts while the live button remains correct.
- [ ] Eight up plus four excluded → Gemini unbenched/stopped shows eight started/up, three excluded and Start fleet · 1; membership/participation changes refresh the same projection. Target retirement withdraws old state rather than inventing zeros.
- [ ] An older late answer cannot overwrite a retired result or newly armed stop plan.
- [ ] Active batch waves, two-press Stop, pending/UNKNOWN/refused results and their reasons retain their behavior.
- [ ] Focused owning units and isolated rendered journeys pass; header text/control remain readable at supported widths.
## Post-Merge Validation

- [ ] Re-read on a named installed candidate carrying the repair. Residual-Owner: #479 (under #477). A source or fixture pass does not retire that installed check.

## Out of Scope

Changing lifecycle admission, stop/start verbs, live presence counts, runtime-generation custody (#674), starting/stopping live peer harnesses for tests, or redesigning the Fleet layout.

## Avoided Traps

Silently treating a batch receipt as current fleet state; mutating historical outcomes instead of deriving a separate current projection; clearing an active wave and cancelling remaining work; hiding UNKNOWN as success; fencing only the text while leaving an old writer active; relabelling stale feedback without retiring it on subsequent actions.

Decision Record impact: none — existing view-state ownership and lifecycle contract preserved.
Structure map: N/A — existing controller layer and established cockpit unit/E2E siblings.

## Related

Parent #477 · views #505 · installed read #479 · completed fleet-button contract #618 / PR #643.

Origin Session ID: 03290248-0760-4e87-b12b-040a5d5dbb14
Retrieval Hint: Fleet header historical start summary stale after roster play stop; FleetBatchController onRosterSettled shared summary slot.

Creation sweeps: latest-open/claim/problem-memory and own-assignment reads refreshed immediately before filing; no equivalent found. Problem recall returned mostly unrelated legacy rows; live source and installed inspection establish the defect. Existing same-surface parent #477 owns the outcome; #596 is observer Activity, not this slot; #657 is dependency security. Historical #618 is completed and stays closed.

Operator clarification (2026-10-14): `8 started · 4 excluded` beside `Start fleet · 1` is compatible with a newly unbenched Gemini, but the live excluded count must be three. The primary header now explicitly targets current state; the earlier historical-only prescription is withdrawn.



## Timeline

- 2026-10-14T10:45:09Z @neo-gpt-euclid assigned to @neo-gpt-euclid
- 2026-10-14T10:45:10Z @neo-gpt-euclid added the `bug` label
- 2026-10-14T10:45:10Z @neo-gpt-euclid added the `agent-os` label
- 2026-10-14T10:45:11Z @neo-gpt-euclid added the `ai` label
- 2026-10-14T10:45:11Z @neo-gpt-euclid added the `design` label
- 2026-10-14T10:45:11Z @neo-gpt-euclid added the `testing` label
- 2026-10-14T10:45:14Z @neo-gpt-euclid added parent issue #477
- 2026-10-14T10:47:49Z @neo-gpt-euclid cross-referenced by #505
- 2026-10-14T11:04:01Z @neo-gpt-euclid changed title from **Retire stale fleet-action feedback after roster lifecycle changes** to **Keep Fleet header counts current after roster lifecycle changes**
- 2026-10-14T11:42:46Z @neo-gpt-euclid cross-referenced by PR #691
- 2026-10-14T12:01:54Z @neo-gpt-euclid referenced in commit `2371582` - "fix(fleet): preserve roster order and refresh captures (#690)"
- 2026-10-14T13:15:34Z @neo-gpt-euclid referenced in commit `1cdeac1` - "fix(fleet): keep count labels together and revalidate captures (#690)"
- 2026-10-14T13:32:08Z @tobiu referenced in commit `afabdef` - "fix(fleet): refresh header counts from current roster (#690) (#691)

* fix(fleet): refresh header counts from current roster (#690)

* fix(fleet): preserve roster order and refresh captures (#690)

* fix(fleet): keep count labels together and revalidate captures (#690)"
- 2026-10-14T13:32:08Z @tobiu closed this issue
- 2026-10-14T13:39:46Z @neo-gpt-euclid cross-referenced by #477
- 2026-10-17T18:29:33Z @neo-fable-clio cross-referenced by #725
- 2026-10-17T19:33:04Z @neo-fable-clio cross-referenced by PR #728

