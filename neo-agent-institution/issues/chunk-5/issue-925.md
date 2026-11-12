---
id: 925
title: Accounts reads the agent registry once at boot and never again
state: OPEN
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-11-12T00:39:18Z'
updatedAt: '2026-11-12T00:48:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/925'
author: neo-opus-grace
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
---
# Accounts reads the agent registry once at boot and never again

## Context

The operator opened Accounts on the installed FM (Institution `6f00d3da` / Brain `7219f4e1` / Engine `c7e19744`, launched 2026-11-11 18:15Z) to add a repository to every seat. The list read `No agents yet. Add the first one with the form.` while the registry held 13 agents and the cockpit showed all 13.

Live reads over the Neural Link, 2026-11-12 00:28Z:
- `agentDefinitions` Store: 0 rows, `isLoaded: false`. `fleetRoster`: 13 rows.
- `AgentOS.view.accounts.Controller`: `agentDefinitionsLoadGeneration: 1`, `fleetTenantsLoadGeneration: 1`, one read each.
- `fleetTenants`: 0 rows, while the registry holds one tenant.
- The panel's `loadAgentDefinitions()` returned `true` and landed all 13 definitions. The Brain answers; the Body never asked again.

The read window, from `main.log` (row 5's H3: a revoked PAT, renewed through Connect; #921 owns the log's wording):

```
18:15:25.045Z HARNESS_BRAIN [fleet] boot phase plane init: 180 ms (refused: plane unreachable (Error))
18:16:59.580Z HARNESS_BRAIN_RETRY after [plane-credential-refused] with a renewed PAT
18:17:00.186Z HARNESS_BRAIN [fleet] boot phase transport: 46 ms (port: 8083)
```

## The Problem

The Accounts panel is a card in the shell's TabContainer, so it is built at boot. Its controller reads `listAgents` and `listTenants` once, from `onComponentConstructed`. A failed read keeps the prior rows, and at boot there are none. Nothing reads again, so a boot before the Fleet serves leaves Accounts empty for the app's lifetime: six hours here. The cockpit recovered on its own because its liveness owner re-polls the roster.

The empty Store is shared, so one failed read costs three surfaces:
1. Accounts lists no agents, and its configuration card has no tenant to offer.
2. The cockpit's agent detail has no definition for its configuration tab: `fleet.detail` reads the same Store, and Accounts is its only hydrator.
3. The empty line says `No agents yet`: a registry that never answered reads as an empty one, beside an open add form.

## The Architectural Reality

Design authority: the sibling form's JSDoc, `AgentOS.view.fleet.instances.AddAgentForm#afterSetMounted`:
> "The form can be built at shell boot, before the Fleet's bridge is wired, so an answer read at construction would be stale."

And #424's terminal predicate: "the product returns to `live` by its own guidance alone".

- `apps/agentos/view/accounts/Controller.mjs`: `onComponentConstructed` calls `loadAgentDefinitions()` and `loadFleetTenants()`. Both already fence stale replies: load generation, the shared write generation, Store identity and lifetime.
- `apps/agentos/view/accounts/Panel.mjs`: `syncDetail()` shows `accounts-empty` whenever the definitions Store has no rows.
- `apps/agentos/view/Viewport.mjs` declares `gridAdapterState` on the root provider: `'cold'` until a source answers, then `'live'` or `'stale'`. `FleetAdmission.admitRoster` writes `'live'` with every admitted roster, and `TargetBinding` resets it to `'cold'` for a new instance. Home already binds it.
- `AgentOS.store.AgentDefinitions` has no inline data, so `isLoaded` stays `false` until a row set lands. `Neo.data.Store#add` sets it, for an empty answer too.

## The Fix

1. `Panel` binds `gridAdapterState`, as Home does. When it turns `'live'` after construction (at boot, after an outage, or for a newly bound instance), the panel runs both controller reads again. The controller's fences discard superseded replies; there is no retry timer.
2. `syncDetail()` shows `No agents yet` only once the definitions Store has loaded. Before that, the line says the Fleet has not answered yet.

## Acceptance Criteria

- [ ] A panel built while the Fleet is unreachable lists the agents and tenants once `gridAdapterState` turns `'live'`, with no reopen and no restart. Unit spec, red on `499339f5`.
- [ ] A `'live'` that does not change the state reads nothing; `'stale'` → `'live'` reads both again. Same spec.
- [ ] An unanswered registry never shows `No agents yet`; an answered empty one does. Same spec.
- [ ] *(post-merge, installed)* `[L4-deferred — operator handoff needed]` Provoke row 5's H3 (a revoked PAT renewed through Connect), then open Accounts: the agents are listed. One receipt on #424.

## Out of Scope

- The add form opening by default when nothing is selected.
- `plane admission unknown` on the add form.
- The E2E specs that call `loadAgentDefinitions` by hand.
- #917, #919 and #921: the same launch walk, other surfaces.

## Avoided Traps

- **Re-read when Accounts mounts** (the form's own pattern): the cockpit's detail stays without definitions until Accounts is opened, and an open Accounts stays empty through an outage.
- **A retry timer in the controller**: a second liveness owner beside the cockpit's.

## Related

#424 (row 5, parent) · #453 (the controller's origin) · #917, #919, #921 (the same walk) · #477 (row 2's truthful states)

Sweeps, 2026-11-12 00:35Z–00:40Z:
- **Live latest-open + closed:** the latest 20 open (57 open, red band; #922..#732). Closed and all-state searches: `Accounts empty agents list boot`, `loadAgentDefinitions OR "No agents yet"`. Read #917, #919, #921 and #453. No equivalent; folds under #424.
- **A2A:** the 30 newest messages, all states, re-checked at 00:40Z. No claim on Accounts.
- **MC:** `Accounts view shows "No agents yet" empty agent list after Fleet Manager update…`, 6 results, no prior decision.
- **Own-assignment:** 4 open (#414, #490, #633, #635), none overlapping.

Retrieval Hint: "Accounts empty after a boot during a Fleet outage, registry read never retried"

🖖 Grace (Claude Opus 5.5, Claude Code)


## Timeline

- 2026-11-12T00:39:19Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-12T00:39:20Z @neo-opus-grace added the `bug` label
- 2026-11-12T00:39:20Z @neo-opus-grace added the `agent-os` label
- 2026-11-12T00:39:20Z @neo-opus-grace added the `ai` label
- 2026-11-12T00:39:27Z @neo-opus-grace added parent issue #424
- 2026-11-12T00:48:31Z @neo-opus-grace cross-referenced by PR #926
- 2026-11-12T01:04:42Z @neo-opus-grace referenced in commit `359a175` - "test(agentos): re-stamp the visual baselines over the Accounts inputs (#925)

Panel.mjs and Controller.mjs are style-owning inputs of the Darwin goldens.
The goldens were re-verified on macOS at this head with no golden changed,
so only the two input entries move and the engine entry stays."
- 2026-11-12T01:15:52Z @neo-opus-grace referenced in commit `7b6d932` - "test(agentos): the isolated boot's Accounts line says the Fleet has not answered (#925)

The isolated E2E has no Fleet, so the Accounts registry read never lands.
Its assertion pinned "No agents yet", the line this ticket retires for a
registry that never answered."

