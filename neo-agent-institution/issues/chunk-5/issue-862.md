---
id: 862
title: Sort the Accounts agent list alphabetically by displayed name
state: CLOSED
labels:
  - enhancement
  - good first issue
  - help wanted
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-11-03T05:24:07Z'
updatedAt: '2026-11-13T14:50:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/862'
author: neo-gpt-euclid
commentsCount: 2
parentIssue: 505
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 2
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-13T14:50:29Z'
---
# Sort the Accounts agent list alphabetically by displayed name

## Context

The operator requested this Accounts-list improvement on 2026-11-03: “this list should better be sorted by agent name.” The supplied selected-state screenshot shows a non-alphabetical agent list. This ticket captures the requested preference; it does not claim a full installed-view sweep.

## The Problem

A predictable order makes an agent easier to find as the list grows. Sorting must use the name the reader sees, and must not move selection to a different agent.

## The Architectural Reality

At Institution `dev 5a608112c1ec3b1a03b7d97ca44d85b820da978c`:

- [Accounts List](https://github.com/neomjs/neo-agent-institution/blob/5a608112c1ec3b1a03b7d97ca44d85b820da978c/apps/agentos/view/accounts/List.mjs#L44) renders `displayName || githubUsername || id`.
- The list binds the provider-owned `AgentDefinitions` Store; its current defaults declare no sorter. Accounts' controller hydrates that Store from registry readback, and the panel retains `selectedAgentId`.
- This is an Institution presentation change using existing Neo Store/Collection sorting, with no registry or Brain API change.

Design authority: operator request quoted above; the requested order is A–Z by the displayed agent name, case-insensitive, with selection preserved.

## The Fix

Apply the ordering through the existing Store/Collection sorting primitive at the agent-definition presentation owner. Use the same label fallback as the renderer, with stable identity as the tie-breaker. Keep the order after initial load, refresh and accepted definition changes. Preserve the provider-owned Store and identity-based selection.

## Contract Ledger

| Surface | Authority | Behavior / fallback | Evidence |
|---|---|---|---|
| Accounts agent list | Operator request; existing row label | Case-insensitive A–Z by `displayName || githubUsername || id`; stable id breaks ties | Mixed-name/fallback ordering coverage and rendered list |
| Selected account | Existing `selectedAgentId` | Reordering retains the selected id and its configuration card | Select, refresh/reorder, assert same agent |

## Acceptance Criteria

- [ ] Initial load and refresh render the agents in case-insensitive A–Z order using the displayed label.
- [ ] Missing display names use the existing username/id fallback; equal names have deterministic order.
- [ ] Adding or changing a definition keeps the order, without changing the selected agent or showing another agent's configuration.
- [ ] Extend the existing Accounts coverage (`test/playwright/unit/apps/agentos/Accounts.spec.mjs` and, where appropriate, `test/playwright/e2e/agentos/AccountsConfigSurface.spec.mjs`); verify the rendered order with representative mixed-case names and an existing selection.

## Out of Scope

Other FM lists, changing displayed names, search/filter controls, grouping by harness, registry persistence/order, or installing/restarting the operator's app.

## Avoided Traps

Do not sort DOM nodes or introduce a hand-mapped parallel row array. Do not sort by a hidden field that differs from the rendered name.

Decision Record impact: none; existing presentation primitive and explicit operator preference. Structure map: N/A — Institution view/Store defaults, no Agent OS service or new module.

Related: #505 (Accounts reading affordances). The Observatory's node-count ordering in closed `#375` is a different surface and remains unchanged.

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Origin Session ID: fe48bcbf-dee0-4a0a-b302-037f7c73e992

Live latest-open + closed sweep: latest 20 open Institution issues and closed `Accounts sort` / all-state `alphabetical` searches at 2026-11-03T05:23Z; adjacent Accounts and Observatory decisions read, no equivalent ordering ticket or exclusion found.
MC sweep: Accounts selector names/findability/order, two queries with three results each; no relevant prior decision returned. KB ticket search found no matching request.
Own-assignment sweep: two open issues, #477 and #657; their state-guidance/security scope does not own this ordering.
A2A claim sweep: latest 30 messages across all read states; no overlapping Accounts sorting claim in the recent window.


## Timeline

- 2026-11-03T05:24:09Z @neo-gpt-euclid added the `enhancement` label
- 2026-11-03T05:24:09Z @neo-gpt-euclid added the `good first issue` label
- 2026-11-03T05:24:09Z @neo-gpt-euclid added the `help wanted` label
- 2026-11-03T05:24:09Z @neo-gpt-euclid added the `ai` label
- 2026-11-03T05:24:49Z @neo-gpt-euclid added parent issue #505
- 2026-11-03T07:10:57Z @riteshyadav33302111-cell cross-referenced by PR #863
- 2026-11-03T08:17:14Z @neo-opus-ada cross-referenced by #864
### @jay23606 - 2026-11-09T12:00:39Z

I'd like to take this one. The sort is well specified, so I can start on the Accounts list ordering and its tests unless someone already has it.

### @github-actions - 2026-11-09T12:00:56Z

Thanks, @jay23606! This one is taken: #863 is open for it.

These `good first issue` issues are open and nobody holds them:

- #869
- #898

The full list [QUARANTINED_URL: github.com] has the rest. Comment on one to claim it.

<sub>An automatic reply. A maintainer follows up. How claims work [QUARANTINED_URL: github.com]</sub>
<!-- claim-responder:@jay23606 -->

- 2026-11-10T19:04:53Z @neo-fable-clio cross-referenced by #907
- 2026-11-13T13:41:35Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-13T13:59:15Z @neo-opus-ada cross-referenced by PR #954
- 2026-11-13T14:21:03Z @neo-opus-ada referenced in commit `8ca340c` - "test(accounts): a browser run proves the rendered order and the kept selection (#862)

Mixed-case labels whose A-Z order is the reverse of their ids, a selected
non-first row, and a registry refresh that moves it. On dev's unsorted store
the first order assertion fails; with the sorter it passes."
- 2026-11-13T14:50:29Z @tobiu referenced in commit `8e9ec43` - "feat(accounts): sort the agent list by displayed name (#862) (#954)

* feat(accounts): sort agent list alphabetically by displayed name (#862)

* refactor(accounts): the store keeps its own order (#862)

The sorter opts into reapplyOnRecordChange, so the engine re-sorts the
Store after an accepted definition edit. The hand-rolled re-sorts in
AgentDefinitions#onRecordChange and Panel#onAgentRosterChange go, as do
the never-deciding id sorter and the Panel's sorter seeding: the app
seats an AgentDefinitions store that declares its sorter. The Controller's
selection restore stays; the hydration spec fails without it.

* test(accounts): restamp the visual inputs for the sorted roster (#862)

All 76 screenshot captures pass unchanged at this head (visual, e2e, e2e:nl);
only the stamp's input identity moves with the apps/agentos sources.

* test(accounts): a browser run proves the rendered order and the kept selection (#862)

Mixed-case labels whose A-Z order is the reverse of their ids, a selected
non-first row, and a registry refresh that moves it. On dev's unsorted store
the first order assertion fails; with the sorter it passes.

---------

Co-authored-by: Ritesh Yadav <riteshyadav33302111@gmail.com>"
- 2026-11-13T14:50:30Z @tobiu closed this issue

