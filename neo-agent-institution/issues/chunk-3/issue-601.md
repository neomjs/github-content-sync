---
id: 601
title: Accounts body gestures start a popup drag
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - design
assignees:
  - neo-gpt-sophie
createdAt: '2026-10-08T00:47:06Z'
updatedAt: '2026-10-09T12:11:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/601'
author: neo-gpt-sophie
commentsCount: 3
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 19465 Handle-scoped sorts still claim body gestures'
blocking: []
closedAt: '2026-10-09T12:11:33Z'
---
# Accounts body gestures start a popup drag

## Context

The operator reported on 2026-10-08 that a small drag anywhere in the Accounts pane opens a popup. This interrupts normal work in the agent list and detail form. The operator explicitly separated this from the popup-close restoration defect.

Observed installed candidate: Institution `fd958fba1e7d492a5ae2b9b283df1f65e5cc1a6e`, Engine `82bc6158444306e0c342e8cda480e77158c9fedb`. Read-only inspection confirms the Accounts wrapper carries `neo-draggable` and its dashboard sort zone accepts `.neo-draggable`.

## The Problem

Body interactions can be admitted as pane movement because the same marker identifies the sortable item and its supposed handle. Clicking, selecting text and using controls must not accidentally detach the view.

Design authority: operator, 2026-10-08: “dragging a view BODY should not be possible => header based dragging wins.”

## The Architectural Reality

Accounts is a legacy `Neo.dashboard.Panel` inside its own `Neo.dashboard.Container`, separate from the cockpit DockLayout. See [Accounts header](https://github.com/neomjs/neo-agent-institution/blob/fd958fba1e7d492a5ae2b9b283df1f65e5cc1a6e/apps/agentos/view/accounts/Panel.mjs) and [dashboard host](https://github.com/neomjs/neo-agent-institution/blob/fd958fba1e7d492a5ae2b9b283df1f65e5cc1a6e/apps/agentos/view/Viewport.mjs). The header explicitly has `neo-draggable`; Engine `src/draggable/container/DragZone.mjs#adjustItemCls` also adds that class to sortable wrappers. Dashboard SortZone defaults its handle selector to the same class.

## The Fix

After Engine prerequisite neomjs/neo#19465 preserves native body interaction, configure the Accounts dashboard's sort zone with a selector that matches only its intended header. Give that header a distinct selector class while retaining its native `neo-draggable` marker. Preserve header-based sorting and tear-out. The [measured selector-only counterexample](https://github.com/neomjs/neo-agent-institution/issues/601#issuecomment-6050929154) shows why the consumer change alone is insufficient.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| Accounts drag admission | Operator decision above; existing dashboard handle selector | Only its header starts pane movement | Body remains ordinary interactive content | Header/host intent in existing JSDoc | Native gesture test with header positive control and body negative controls |

Decision Record impact: none; restores the requested interaction boundary using existing Engine configuration. Structure-map command is not hosted in Engine; source placement stays in existing Accounts/Viewport files, with no new service or module.

## Acceptance Criteria

- [ ] A drag beginning in body whitespace, list text, detail text or form controls does not start the dashboard drag or open a popup.
- [ ] A deliberate Accounts header drag still starts pane movement and can tear out.
- [ ] Body selection, scrolling and controls remain usable.
- [ ] Regression coverage fails on the broad body selector and passes with the header boundary. [L3-deferred — operator handoff needed] An installed receipt records the actual candidate separately from source tests; the next-candidate witness remains with Sophie under #505.

## Post-Merge Validation

On the next agreed installed candidate, Sophie repeats the Accounts body/header gestures and records the actual Product and Engine inputs on #505. This requires the merged consumer and Engine pin; source/browser evidence does not certify the installed app. No new harness restart or app replacement is authorized by this ticket alone.

## Out of Scope

Popup-close restoration/duplication; generic DockLayout changes; removing Accounts popout capability.

## Related and discovery

Related: #12 and #505. The separately tracked Engine popup-close defect must remain independently testable with a valid header drag.

Live latest-open sweep: latest 20 created-descending Institution and Engine issues at 2026-10-08T00:46Z, no equivalent. All-state latest 30 A2A messages: no competing Accounts claim. Historical org search `Accounts popup`: no match. MC sweep `Accounts pane body dragging popup close duplicated view`: three results, earlier popup/dock incidents but no matching header-boundary decision. Own-assignment sweep: Institution none; Engine only `#19446`, a different surface.

Owner: Sophie. Native blocked-by edge: neomjs/neo#19465. The Engine prerequisite merged in neomjs/neo#19466 at `d75cc685`. The consumer includes that exact Engine pin; installed FM acceptance remains explicitly tracked above.

Origin Session ID: e6ce4d70-a7ff-454e-996d-e7c25efdf4cf

Retrieval Hint: "Accounts body drag header-only handle popup"


## Timeline

- 2026-10-08T00:47:07Z @neo-gpt-sophie added the `bug` label
- 2026-10-08T00:47:07Z @neo-gpt-sophie added the `agent-os` label
- 2026-10-08T00:47:07Z @neo-gpt-sophie added the `ai` label
- 2026-10-08T00:47:08Z @neo-gpt-sophie added the `design` label
- 2026-10-08T00:47:54Z @neo-gpt-sophie cross-referenced by #19462
- 2026-10-08T00:50:38Z @neo-gpt-sophie cross-referenced by #12
- 2026-10-08T01:58:29Z @neo-gpt-sophie cross-referenced by PR #19464
- 2026-10-08T02:17:23Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-10-08T02:26:11Z @neo-gpt-sophie cross-referenced by #19465
### @neo-gpt-sophie - 2026-10-08T02:28:10Z

The selector-only candidate fails the body-interaction requirement in a real browser. With a distinct header selector, a body press still sets `neo-drag-active` and leaves text selection empty, although the App worker correctly refuses sorting (`currentIndex:-1`, no window drag). Removing only the automatic outer `neo-draggable` marker restores selection (`Select these ordinary `), and the same header still tears out.

Engine prerequisite: neomjs/neo#19465, now self-assigned and in implementation. Its scope is the existing native-marker projection, preserving whole-item/root-handle behavior and the legitimate selection guard. The Accounts header/config change and installed receipt stay here; this finding prevents shipping a no-popup fix that still breaks body interaction.

Evidence: local configured legacy-dashboard fixture; original installed FM remains unchanged. I now own this leaf's follow-through, replacing the earlier unowned disposition.

- 2026-10-08T02:32:20Z @neo-gpt-sophie marked this issue as being blocked by #19465
- 2026-10-08T02:56:38Z @neo-gpt-sophie cross-referenced by PR #19466
### @neo-gpt-sophie - 2026-10-08T03:24:40Z

Implementation sequencing is now concrete: Engine [#19466](https://github.com/neomjs/neo/pull/19466) delivers the native prerequisite from `#19465`; its 38 checks are green and Vega holds the GitHub review seat. The consumer patch is prepared against Institution `68bd58cb`: retain `neo-draggable` on the Accounts header, add `fm-accounts-drag-handle`, and set the Accounts dashboard's `sortZoneConfig.dragHandleSelector` to that distinct class.

Emmy has offered to integrate that patch under her existing mapped checkout identity after the prerequisite merges and the checkout's source/custody boundary is verified. Sophie retains this outcome and the installed acceptance. The patch is not applied, and the eventual Engine pin plus Institution tests remain part of that integration. The source setup was not bypassed by creating another checkout or borrowing a peer's identity.

- 2026-10-08T04:22:45Z @neo-gpt-sophie cross-referenced by #602
- 2026-10-08T15:07:22Z @neo-gpt-emmy cross-referenced by #606
### @neo-gpt-sophie - 2026-10-09T04:47:03Z

Source validation on Product `32627ab` plus the Accounts handle configuration and Engine `d75cc685`:

- `AccountsConfigSurface.spec.mjs` now drives body whitespace, list/detail text, input and wheel scrolling with native mouse events, checking both the native drag guard and App-worker sort state. The header still starts sorting and opens a real popup; its DOM id and preserved input bind it to the original live Accounts instance. Final focused run passes.
- The test fails on the original implementation. With the new Engine still installed, restoring only the broad `.neo-draggable` selector also fails at the first body press. The final source restores the distinct header selector.
- Required full headed NL comparison: **71 passed / 6 failed before → 72 passed / the same 6 failed cases after**. Baseline failures are the AgentCard, DrillRoundTrip, GoldenPath, Memories and NavFamily screenshot arms plus the CockpitStateWalkthrough roster census. This is a baseline comparison, not a green full-suite claim.
- Full Darwin visual recapture: **43 passed / 2 failed; no PNG changes**. The existing setup witness-row test expects three preset controls hidden by the new Create flow and also fails on the prior Engine. A Perspectives-pressed failure in the pane-head census did not reproduce: prior Engine control passed and the proposed pin passed **3/3** isolated trials. That single transient remains disclosed; no causal Engine defect is asserted.

The Engine pin advances exactly one merged commit, neomjs/neo#19466. Current source uses the existing `sortZoneConfig.dragHandleSelector` mechanism and preserves `neo-draggable` on the header. Installed acceptance remains the separately recorded next-candidate witness under #505; no installed app or active peer harness was changed.

- 2026-10-09T04:49:10Z @neo-gpt-sophie cross-referenced by PR #627
- 2026-10-09T06:53:27Z @neo-opus-grace cross-referenced by #638
- 2026-10-09T12:11:33Z @tobiu referenced in commit `9ce0576` - "fix(agentos): restrict Accounts dragging to its header (#601) (#627)"
- 2026-10-09T12:11:33Z @tobiu closed this issue

