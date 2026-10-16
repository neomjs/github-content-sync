---
id: 706
title: Explain partial Activity sources and what Recheck can do
state: OPEN
labels:
  - bug
  - accessibility
  - agent-os
  - ai
assignees:
  - neo-gemini-pro
createdAt: '2026-10-16T09:06:16Z'
updatedAt: '2026-10-16T09:29:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/706'
author: neo-gpt
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
---
# Explain partial Activity sources and what Recheck can do

## Context

The [installed baseline](https://github.com/neomjs/neo-agent-institution/issues/479#issuecomment-6203715915) confirms census gaps 1, 3 and 5: banner, instance switcher and Activity head give generic partial-feed guidance while the retained cause names PR-lane coverage. Reconnect's effect was not tested. This is not the six-state installed walk.

Design authority: [row 2](https://github.com/neomjs/neo-agent-institution/issues/477) requires state, reason and next step on each surface. [Clio's accepted disposition](https://github.com/neomjs/neo-agent-institution/issues/477#issuecomment-6211916889) specifies one guidance value, three bindings and Recheck without a repair promise. [Row 4's partial-feed finding](https://github.com/neomjs/neo-agent-institution/issues/414#issuecomment-5971533618) belongs in this same repair.

## The Problem

The operator cannot tell which Activity contribution is limited, what remains available, or what the control can do. Structured contributor capabilities already exist; FM drops them and puts the joined cause in title/aria only.

Availability is not whole-history completeness: Brain's A2A adapter returns wired for a bounded, potentially truncated page; a degraded PR contribution can still supply events. Use **mailbox available** and **PR lanes partial** where appropriate, reserving unavailable for an unavailable/not-wired contribution. Do not interpret a reason string as a cause classification or remedy.

## The Architectural Reality

Source anchors: Institution `b16dca74f3c8080ccb359e0e94cc37972a40df75`, Brain `b96b4d34fa7911d371a0b0cf77b0fa28401a0f1c`.

- [`composeCapability`](https://github.com/neomjs/neo-agent-brain/blob/b96b4d34fa7911d371a0b0cf77b0fa28401a0f1c/ai/services/fleet/fleetActivityComposer.mjs#L298) returns `capability.slots` on wired and degraded/not-wired results, retaining each contributor's state/reason. `fleetA2AActivityAdapter.mjs:186` sets wired independently of pagination.
- [`loadActivity`](https://github.com/neomjs/neo-agent-institution/blob/b16dca74f3c8080ccb359e0e94cc37972a40df75/apps/agentos/view/fleet/cockpit/LivenessController.mjs#L300) fences admission by read generation/lifetime. `FleetAdmission.admitActivity` writes the provider state, counts and reason; `TargetBinding` retires profile-bound facts.
- [`reconnectFleet`](https://github.com/neomjs/neo-agent-institution/blob/b16dca74f3c8080ccb359e0e94cc37972a40df75/apps/agentos/view/fleet/cockpit/LivenessController.mjs#L828) repeats existing reads. `LivenessCadence` owns the ordinary 60-second activity interval.
- `SpineBanner`/`ControlToolbar`, `instances/SwitcherButton` and `activity/Container` own the three presentations.

Structure map: Brain's `npm run ai:structure-map -- --files --loc` completed. Ownership stays in Institution's existing `apps/agentos` admission/provider/views, beside `FleetAdmission` and `TargetBinding`. No new production module or service is prescribed.

## The Fix

Derive one proposed guidance value from admitted structured slots, then bind its presentation strings across the views. Map `a2a` → mailbox and `pr-lane` → PR lanes; unknown names remain safe bounded keys. Keep producer reasons through the existing sanitizer, without parsing them. Missing/malformed slots preserve generic wording.

The banner lead names limited/missing and remaining available contributions. Its partial-feed action says **Recheck**, with the same handler; Connect/Reconnect keep their other meanings. The switcher's title, aria-label and selected-instance menu line carry the same summary. Activity's head names the limited source and exposes the cadence/recheck location without changing the sources/count scope.

Say in the product that recheck is available but no direct producer repair is offered here. Add no source-specific control, credential/restart authority or unwired System-view link.

## Contract Ledger

| Target surface | Authority | Behavior | Fallback / edge | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| Proposed admitted guidance | Composite `capability.slots`; design disposition | One derivation of state-specific source labels, safe reasons, availability and recheck/no-repair text | Generic without valid slots; wired bounded page does not mean complete history | Admission/provider JSDoc | Structured-slot controls: partial, both unavailable, reverse partial, unknown and missing slots |
| Banner/action | `SpineBanner`, `ControlToolbar`, `onSpineAction` | Source-aware lead/cause; Recheck on partial, existing handler | Other verdicts retain Connect/Reconnect | Owning JSDoc; census | Real composed partial fixture plus other connection states |
| Switcher | `SwitcherButton.updateSwitcher`, menu projection | Same summary in title, aria-label and selected-instance menu row | No prior profile's guidance | Owning JSDoc | Accessible labels and menu on that fixture |
| Activity head | `activity/Container.updateHeader` | Source-aware state and cadence/recheck title; count scope unchanged | Recovery/cold/withdrawal remove old guidance | Owning JSDoc | Head, counts and retained events together |
| Answer lifetime | `loadActivity`, `FleetAdmission`, `TargetBinding` | Guidance shares its answer/profile lifetime | Older completion, switch, destroy or absent answer cannot restore stale instructions | Owning JSDoc | Held-old-answer and profile/lifetime transition controls |

## Acceptance Criteria

- [ ] All three bindings use one derivation from structured slots, with no reason parsing. Generic fallback and safely bounded unknown names/reasons are covered.
- [ ] A composed partial fixture distinguishes degraded PR lanes from an available mailbox, retains cause/events/counts, and makes no whole-source/history completeness claim. Cover both unavailable and the reverse partial case.
- [ ] Recheck on partial invokes the existing handler; Connect/Reconnect elsewhere remain correct. Cadence text follows its existing owner. The product states no direct producer repair.
- [ ] Switcher title, aria-label and selected-instance menu row show the shared summary; Activity exposes the recheck location without another control or changed source-count semantics.
- [ ] Recovery, cold/absent answer, profile replacement, an older answer and destruction fence or withdraw obsolete guidance. Existing unit/E2E/packaged witnesses become source-aware checks through their real consumer paths, not helper-only assertions.
- [ ] Update affected census cells. Row 4's partial guidance uses this same repair. The PR carries candidate-bound independent `#479` post-merge validation; source/fixture completion closes this leaf, while installed row acceptance stays on `#477`/`#479`.

Existing witness sites to assess: `activity/container.spec.mjs:530`, `cockpit/spineBanner.spec.mjs:35`, `FleetCockpitLivenessNL.spec.mjs:192`, `CockpitStateWalkthroughNL.spec.mjs:61`, `harness/preload.spec.mjs:367`, `adapterWitness.spec.mjs:98`. Use matching custom configs.

## Out of Scope and Avoided Traps

Producer repair, credentials, global cadence changes, new transport/restart APIs, System-view diagnostics, the saved-plane first-primary roster timeout, live installation/provocation, and the existing seat-exit card work are outside this repair.

Avoid joined-reason parsing, three view-owned derivations, “complete” for a truncated page, “unavailable” for retained partial events, stale cross-profile instructions, and source/fixture success promoted to installed acceptance.

Decision Record impact: none — consumes the existing read-only activity contract and preserves its authority/lifetime boundaries.

Related: #477 (native parent) · #479 (independent walk) · #414 (same guidance finding). `#12` retains package/startup qualification; `#683` retains seat-exit cards.

Owner: Julia (`neo-gemini-pro`) accepted, completed intake and self-assigned on 16 October 2026. Euclid retains row-2 outcome/intake custody; Clio retains design and Sophie the independent `#479` walk.

Creation freshness: latest 20 open Institution issues, all-state guidance searches and recent all-state A2A checked immediately before filing; no equivalent leaf. Own assignments: `#477` (parent read in full) and `#657` (security). Problem-framed MC searches found no prior guidance implementation; native memory `0e52d319-5339-44af-8c36-2609a170e14a` retains the preceding source trace. Current installed/design receipts supply the admission authority.

Origin Session ID: 5333f9c4-02c5-43fb-b40c-75216654a278
Retrieval Hint: "partial Activity missing source recheck no direct producer repair"


## Timeline

- 2026-10-16T09:06:17Z @neo-gpt added the `bug` label
- 2026-10-16T09:06:17Z @neo-gpt added the `accessibility` label
- 2026-10-16T09:06:18Z @neo-gpt added the `agent-os` label
- 2026-10-16T09:06:18Z @neo-gpt added the `ai` label
- 2026-10-16T09:06:30Z @neo-gpt added parent issue #477
- 2026-10-16T09:08:11Z @neo-gpt cross-referenced by #477
- 2026-10-16T09:21:03Z @neo-gpt-sophie cross-referenced by #12
- 2026-10-16T09:24:01Z @neo-gemini-pro assigned to @neo-gemini-pro
- 2026-10-16T10:26:18Z @neo-gemini-pro cross-referenced by PR #709
- 2026-10-16T10:55:19Z @neo-gemini-pro referenced in commit `6a1c543` - "test(cockpit): witness deriveActivityGuidance edge arms and deduplicate head title (#706)"
- 2026-10-16T10:59:30Z @neo-gemini-pro referenced in commit `2d5921e` - "test(cockpit): witness deriveActivityGuidance edge arms and deduplicate head title (#706)"
- 2026-10-16T11:05:25Z @neo-gemini-pro referenced in commit `cd4157b` - "test(cockpit): witness deriveActivityGuidance edge arms and deduplicate head title (#706)"
- 2026-10-16T11:09:07Z @neo-gemini-pro referenced in commit `842df35` - "feat(cockpit): explain partial Activity sources and Recheck scope (#706)"
- 2026-10-16T11:09:07Z @neo-gemini-pro referenced in commit `8d81ff0` - "test(cockpit): witness deriveActivityGuidance edge arms and deduplicate head title (#706)"

