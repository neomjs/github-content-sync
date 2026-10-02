---
id: 465
title: Move inspector config and aging into its view controller
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - refactoring
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-02T18:44:58Z'
updatedAt: '2026-10-02T19:48:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/465'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 24
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 418 The roster card''s lane line and the detail''s lane pane read the roster row''s lane stamp'
blocking: []
closedAt: '2026-10-02T19:48:06Z'
---
# Move inspector config and aging into its view controller

## Context

The #42 [view-layer matrix](https://github.com/neomjs/neo-agent-institution/issues/42#issuecomment-5956656161) identifies the inspector as the next definition-lifecycle seam after Accounts, delivered by #463. This is a bounded application of #24 Law 2, not a runtime-bug claim or a size-only split.

## The Problem

At merged dev `155ec4c`, `apps/agentos/view/fleet/detail/Container.mjs` owns child config-intent wiring (`onConstructed`), the shared definitions Store's subscription lifecycle, `onConfigIntent` and the recurring `startFreshnessAging` scheduler. Moving only the small forwarding handler would add a class without moving a coherent responsibility. The repair is the inspector's combined workflow/lifecycle ownership.

## The Architectural Reality

`Neo.controller.Component.onComponentConstructed` provides the point where child references exist; the component destroys its controller before its children. The existing Accounts, roster and tasks controllers establish this role and placement. The Viewport owns the definitions Store, and `ConfigIntentRoundTrip` owns cross-surface write ordering. The inspector joins `FleetAgent.agentId` to `AgentDefinition.id`; its view renders the resulting record and freshness state.

The current config sink resolves the card lazily. No late-response bug is asserted from static source. Valid canonical writes may still land in a surviving provider Store after an inspector closes; a destroyed inspector must receive no status paint.

## The Fix

Add `view/fleet/detail/Controller.mjs` extending `Neo.controller.Component`. Move child intent subscription, definitions-Store subscription lifecycle and dispatch, config round-trip ownership, and the existing freshness scheduler into it. Keep composition, public input configs, the Store-to-card rendering join and formatting in Container. Preserve `now`, `freshnessRefreshMs`, the clock source and the existing record/no-record scheduling behavior. Do not add a provider.

Retain a view delegate only for a demonstrated production caller. The current repository-wide search finds `onConfigIntent` called directly only by its unit spec, and `startFreshnessAging` only internally. Recheck that at pickup.

## Contract Ledger

| Surface | Authority | Behavior | Fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| Config intent/readback | ConfigIntentRoundTrip | same shared Store, ordering and owner-scoped status channel | refusal or unavailable bridge leaves records unchanged; dead inspector gets no paint | controller summaries | real child-event, refusal, cross-owner and late-result arms |
| Definition join/subscriptions | current agentDefinitions lifecycle | matching id seats the exact shared record; load/mutate/recordChange update presentation | absent/removed definition clears the card; old Store detaches | controller + view hook summaries | add/replace/remove, same-record refresh and Store-switch controls |
| Freshness scheduler | current startFreshnessAging + now/freshnessRefreshMs | same cadence and rendering, owner-lifetime teardown | no record skips rendering but retains existing scheduling behavior | scheduler summary | controlled-clock, empty-record and destroy arms |
| Docked/vesseled inspector | existing composition and phase-independent seating | same component, definitions and status channel across render targets | no new ancestor lookup or copied Store | class summary | served drill/config journey and current visual receipts |

## Acceptance Criteria

- [ ] The controller owns the combined intent/subscription/scheduling lifecycle; those workflow bodies are removed from Container without forwarding copies.
- [ ] Real component/controller tests preserve canonical configure/refusal/supersession behavior, and prove that late accepted/rejected results cannot paint a destroyed inspector while shared Store admission stays unchanged.
- [ ] Definition joins and Store replacement preserve exact record identity and listener teardown across add, replace, remove, reload and same-record changes.
- [ ] Freshness continues to age without a record mutation, respects the injected clock, preserves no-record behavior and stops after owner destruction.
- [ ] The served inspector journey and Darwin visual receipts pass on the final merged dependency base, with the input stamp consistent and no unexplained pixel change.

## Out of Scope

New pane producers, lane projection/card aging, changing time semantics or introducing a shared clock, selection-provider consolidation, Brain contracts, global ConfigIntentRoundTrip policy, and the stale 271px screenshot repair already owned by #461.

## Avoided Traps

No one-method controller shell. No controller-local Store or new provider. No modification of shared admission policy to suppress a valid canonical response. No ancestor-only controller lookup that changes ownership when the pane is vesseled.

Decision Record impact: none — applies the parent epic's existing controller/provider law.

Related: #24 (parent), #42 (measurement), #453 (precedent), #418 (ordered prerequisite).
The native blocked-by edge to #418 preserves Sophie's current inspector projection work. Her A2A `d07f72b2-2401-42d5-b4f3-05b3bf27bb9b` confirms no configuration/lifetime move in that lane.

unowned-rationale: ordered repair leaf from the active measurement lane; implementation starts after #418 closes and current source is revalidated.

Structure map: N/A for the Brain-hosted map runner in this consumer; placement is the established view-root ComponentController role. Structural fast-path: `view/fleet/detail/Controller.mjs` follows `view/accounts/Controller.mjs` and `view/fleet/roster/Controller.mjs`, with no novel directory role.

Sweeps: live latest 20 open Institution issues and recent all-state A2A checked immediately before filing on 2 October 2026; no equivalent. Exact open/all-state inspector/detail-controller searches found none. MC problem-noun recall returned unrelated startup records, not a prior ruling. Own-assignment sweep found only #42, the measurement parent, whose body and current matrix were read. No duplicate repair is hidden in #453, whose scope explicitly excludes the inspector.

Origin Session ID: 8d1cf4b5-75d2-4880-8358-873e0ac47fe0
Retrieval Hint: inspector onConfigIntent startFreshnessAging agentDefinitions controller lifecycle


## Timeline

- 2026-10-02T18:44:59Z @neo-gpt-emmy added the `enhancement` label
- 2026-10-02T18:44:59Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-02T18:45:00Z @neo-gpt-emmy added the `ai` label
- 2026-10-02T18:45:00Z @neo-gpt-emmy added the `refactoring` label
- 2026-10-02T18:45:23Z @neo-gpt-emmy added parent issue #24
- 2026-10-02T18:45:25Z @neo-gpt-emmy marked this issue as being blocked by #418
- 2026-10-02T18:46:02Z @neo-gpt-emmy cross-referenced by #42
- 2026-10-02T19:13:35Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-02T19:34:24Z @neo-gpt-emmy cross-referenced by PR #466
- 2026-10-02T19:48:06Z @tobiu referenced in commit `2303209` - "fix(agentos): give inspector workflows a lifecycle owner (#465) (#466)"
- 2026-10-02T19:48:07Z @tobiu closed this issue

