---
id: 161
title: An accepted agent definition never refreshes the roster
state: CLOSED
labels:
  - bug
  - ai
  - regression
assignees:
  - neo-fable-clio
createdAt: '2026-09-18T15:30:30Z'
updatedAt: '2026-09-18T15:55:24Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/161'
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
closedAt: '2026-09-18T15:55:24Z'
---
# An accepted agent definition never refreshes the roster

## Context

Surfaced by #160: `AccountsConfigSurface.spec.mjs` ran nowhere, and once its first rot layer was repaired it stopped at line 129 — after "Add agent" succeeds (status "Agent added", the definition is in the registry), the Fleet tab shows no `.fm-agent-card` for the new resident. The failure's DOM snapshot reads "Fleet server offline — showing the static roster". Measured on dev@a94bbd1183 with the Brain runtime root bound.

## The Problem

`Viewport#onAgentDefinitionAccepted` is the one route from an accepted definition to the roster: both add-agent surfaces end there (Accounts, `Viewport.mjs:175`; the cockpit's define-agent tool, `cockpit/Container.mjs:230`, both `'up.onAgentDefinitionAccepted'`). It asks the cockpit for a re-poll:

```js
if (typeof cockpit?.loadRoster !== 'function') {
    return false
}

await cockpit.loadRoster();
```

`AgentOS.view.fleet.cockpit.Container` has no `loadRoster`. The method lives on `LivenessController` (`:295`) since the cockpit core was rebuilt on 2026-08-29 (#50); the Container's class doc still links `{@link #loadRoster}` (`Container.mjs:76`). So the guard answers `false` on every accepted definition and nothing re-polls. A live cockpit hides it — the liveness cadence picks the resident up a poll later; a cockpit that booted against the fail-closed bridge never does, which is the spec's case.

The unit arm that owns this contract is green against a stub: `ViewportController.spec.mjs:111-124` hands the handler `{loadRoster: async () => …}` — a method the real class lost. The only witness over the real classes ran nowhere (#160).

## The Architectural Reality

- `apps/agentos/view/Viewport.mjs:195-221` — the handler; `getReference('fleet-cockpit')` is the cockpit Container.
- `apps/agentos/view/fleet/cockpit/LivenessController.mjs:295` — `loadRoster()`, the sanctioned idempotent, fail-closed re-poll; the Container already reaches it as `controller.loadRoster()` (`Container.mjs:727`), and the e2e harness as `controller.loadRoster` (`authenticatedFleetHarness.mjs#reloadRoster`).
- `apps/agentos/view/fleet/cockpit/Container.mjs` is at 982 of its 1000 lines.

## The Fix

The Container gets back the public re-poll its class doc promises — `loadRoster()` delegating to its controller — so a parent view calls a child's method and never reaches into another view's controller. The unit arm pins its stub to the real class (`typeof CockpitContainer.prototype.loadRoster === 'function'`), so a stub can no longer answer for a method the class does not have.

## Acceptance Criteria

- [ ] An accepted definition re-polls the roster through the real cockpit Container: `Viewport#onAgentDefinitionAccepted` resolves `true` against the real class's prototype, red on dev.
- [ ] The stub in the handler's unit arm is tied to the real Container's surface.
- [ ] `AccountsConfigSurface` passes line 129 on a host with the Brain runtime root (outside CI; the spec itself lands green with #160).
- [ ] `Container.mjs` stays under its line limit.

## Out of Scope

The battery selection and the spec's other repairs (#160). Why a cockpit that booted fail-closed stays offline until something re-polls — by design today, the bearer arrives later.

## Related

#160 (the orphaned witness); #50 (the rebuild that moved `loadRoster`).

Live latest-open sweep: all 18 open issues of neomjs/neo-agent-institution at 2026-09-18T15:30Z — none equivalent. A2A in-flight claim sweep: the last hour carries no claim on the Viewport or the accepted-definition route. Memory Core rationale sweep (`query_raw_memories`, the symptom's nouns): the 2026-07 define-agent design placed the refresh at this handler; nothing records the route as retired or re-decided. Own-assignment sweep: #160 (the witness), #127, #129, #10 — other surfaces.

Origin Session ID: 1ef6c04a-10f6-4977-ad7d-0e2c7f343c59
Retrieval Hint: "onAgentDefinitionAccepted cockpit loadRoster Container lost method stub vacuous"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 1ef6c04a-10f6-4977-ad7d-0e2c7f343c59


## Timeline

- 2026-09-18T15:30:30Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-18T15:30:32Z @neo-fable-clio added the `bug` label
- 2026-09-18T15:30:32Z @neo-fable-clio added the `ai` label
- 2026-09-18T15:30:32Z @neo-fable-clio added the `regression` label
- 2026-09-18T15:34:00Z @neo-fable-clio cross-referenced by PR #162
- 2026-09-18T15:46:37Z @neo-fable-clio cross-referenced by #163
- 2026-09-18T15:55:24Z @tobiu referenced in commit `412b451` - "Merge pull request #162 from neomjs/agent/161-accepted-definition-repolls

fix(fleet): an accepted agent definition re-polls the roster through the cockpit (#161)"
- 2026-09-18T15:55:25Z @tobiu closed this issue
- 2026-09-18T16:03:26Z @neo-fable-clio cross-referenced by PR #165
- 2026-09-18T16:10:30Z @neo-fable-clio cross-referenced by #129
- 2026-09-18T16:58:52Z @neo-fable-clio cross-referenced by PR #167
- 2026-09-19T10:32:05Z @neo-fable-clio cross-referenced by #170
- 2026-09-19T13:10:56Z @neo-fable-clio cross-referenced by PR #173

