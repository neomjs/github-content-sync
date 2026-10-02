---
id: 453
title: Accounts registry workflows move to their view controller
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - refactoring
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-02T16:40:07Z'
updatedAt: '2026-10-02T18:32:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/453'
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
blockedBy: []
blocking: []
closedAt: '2026-10-02T18:32:42Z'
---
# Accounts registry workflows move to their view controller

## Context

The #42 [current-tree matrix](https://github.com/neomjs/neo-agent-institution/issues/42#issuecomment-5956656161) identifies Accounts as a concrete #24 Law 2 repair: its panel still owns canonical reads and configuration round trips. This leaf is one behavior-preserving Accounts cut, not a controller added to every rendering component.

## The Problem

At dev `87e4f1c`, `apps/agentos/view/accounts/Panel.mjs` is 624 lines. Its `loadAgentDefinitions` and `loadFleetTenants` read the registry bridge and enforce stale-read fences; `onAgentConfigIntent` and `onAgentReposIntent` drive `ConfigIntentRoundTrip`; `upsertPublicAgentDefinition` validates and writes the shared store. These are lifecycle/IO responsibilities inside the rendering class. The same file also correctly owns selection presentation, child cards and local save-status rendering. Moving the whole class or splitting by size would lose that distinction.

## The Architectural Reality

The Viewport provider owns the shared AgentDefinitions/FleetTenants stores. They must stay the same instances, not become controller-local copies. `ConfigIntentRoundTrip` owns the store-wide write generation and per-owner supersession; its separate config-card and repo-card owner tokens must retain their meaning. A definition read started before a newer accepted write cannot overwrite it. `loadAgentDefinitions` is also an existing Neural-Link/E2E entrypoint.

Prescription checked: `Neo.controller.Component` owns component lifecycle and reference access. The adjacent `view/fleet/roster/Controller.mjs` and `view/fleet/tasks/Controller.mjs` establish the view-root controller pattern. The new `view/accounts/Controller.mjs` is another instance of that role; no new service or provider is warranted.

## The Fix

Create the Accounts view controller beside Panel/List. Move registry reads, generation ownership, definition acceptance/upsert and configuration/repository round-trip orchestration into it. The panel retains composition, store bindings, local selected row and presentation/status sinks. Route child intents through the controller and detach listeners on destruction. Retain only necessary documented view entrypoint delegates where current external callers need them; do not duplicate workflow bodies.

## Contract Ledger

| Surface | Authority | Behavior | Edge case | Docs | Evidence |
|---|---|---|---|---|---|
| Registry reads | current Panel loaders | controller owns read/fence; same provider stores receive accepted rows | failed, malformed, superseded or replaced-store reads retain prior truth | controller JSDoc | Accounts stale-read and store-replacement controls |
| Config/repo writes | ConfigIntentRoundTrip | same persisted readback, distinct owner/status channels | old response cannot paint a newer request or another visible agent | controller + public delegate JSDoc | existing rejection/supersession controls |
| Accepted definition | AddAgentFlow readback and Viewport event seam | upsert same public definition and re-fire accepted event once | invalid definitions do not mutate; add form keeps outcome | handler JSDoc | accepted/invalid event controls |
| View entrypoint | existing AccountsConfigSurface callers | loadAgentDefinitions remains callable, as a thin delegate if retained | no second implementation | delegate JSDoc | existing served journey |

## Acceptance Criteria

- [ ] A component controller owns the registry reads and writes; their implementations are removed from Panel, with only required delegates retained.
- [ ] Failed/malformed/stale reads and configure-during-read preserve the existing store truth and generation semantics.
- [ ] Config and repository status remain independent, keyed to the original agent; late results cannot affect a new selection or destroyed owner.
- [ ] Accepted definition, form retention and Viewport roster refresh semantics remain unchanged.
- [ ] Existing unit and AccountsConfigSurface coverage passes through real component/controller instances; visual input stamp is refreshed without accepting an unexplained visual change.

## Out of Scope

The shared add form and its GitLab work in #448; the inspector's separate controller cut; selected-agent/cockpit provider consolidation; Brain contracts; design changes; adding a provider just for local display state.

## Avoided Traps

No store singleton or copied provider state. No change to ConfigIntentRoundTrip's central admission policy. No size-only slicing. A thin public delegate is permitted only for a demonstrated caller, not as a second layer of forwarding everywhere.

Decision Record impact: none — applies #24's established controller/provider law.

Related: #24 (parent) · #42 (matrix and evidence) · #448 (neighbor, separate scope).

unowned-rationale: a scoped repair leaf produced by the measurement lane; FM v1's setup/pin consumers remain the current delivery priority. Any peer may self-select after checking the current Accounts source and in-flight fixtures.

Sweeps: live latest 20 open Institution issues and all-state recent A2A checked immediately before filing (2 October, 16:40 UTC); no Accounts controller repair or claim. Own assigned #42 owns measurement, #451 owns a dependency pin. Query on Accounts loaders/round-trip ownership returned unrelated memories, not a prior decision. Structure map and exact source census executed; sibling structural fast-path uses the existing view-root ComponentController pattern.

Origin Session ID: 3acb1755-5285-4f3a-a74a-dae637bb629d
Retrieval Hint: Accounts Panel loadAgentDefinitions loadFleetTenants ConfigIntentRoundTrip controller split

## Timeline

- 2026-10-02T16:40:09Z @neo-gpt-emmy added the `enhancement` label
- 2026-10-02T16:40:09Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-02T16:40:09Z @neo-gpt-emmy added the `ai` label
- 2026-10-02T16:40:09Z @neo-gpt-emmy added the `refactoring` label
- 2026-10-02T16:40:50Z @neo-gpt-emmy added parent issue #24
- 2026-10-02T16:40:52Z @neo-gpt-emmy cross-referenced by #24
- 2026-10-02T16:40:53Z @neo-gpt-emmy cross-referenced by #22
- 2026-10-02T16:42:46Z @neo-gpt-emmy cross-referenced by #42
- 2026-10-02T17:58:21Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-02T18:19:21Z @neo-gpt-emmy cross-referenced by PR #463
- 2026-10-02T18:32:42Z @tobiu referenced in commit `155ec4c` - "fix(agentos): give Accounts registry workflows a lifecycle owner (#453) (#463)"
- 2026-10-02T18:32:43Z @tobiu closed this issue
- 2026-10-02T18:44:59Z @neo-gpt-emmy cross-referenced by #465
- 2026-10-02T19:44:06Z @neo-opus-grace cross-referenced by PR #466

