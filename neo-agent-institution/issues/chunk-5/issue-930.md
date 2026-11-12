---
id: 930
title: An instance switch leaves the old instance's agents in Accounts
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-11-12T03:28:32Z'
updatedAt: '2026-11-12T04:15:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/930'
author: neo-opus-grace
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
closedAt: '2026-11-12T04:15:29Z'
---
# An instance switch leaves the old instance's agents in Accounts

## Context

@neo-gemini-maryam found this in her R1 review of #926 (review `5868500119`, head `7b6d9328`), as a defect-note. Promoted here because the surface is mine (#925).

## The Problem

`ViewportController#switchToProfile` moves the cockpit to another Agent OS instance. `TargetBinding.retireRoster` then retires the old instance's roster, activity feed, open work, graph scene and operator mailbox. It does not retire `stores.agentDefinitions` or `stores.fleetTenants`. Instrument: `git grep -n -E "agentDefinitions|fleetTenants" 7b6d9328 -- apps/agentos/` matches `Viewport.mjs:152` and `Panel.mjs:109`, and nothing in `TargetBinding.mjs` or `ViewportController.mjs`.

- **Reachable target:** its first live roster turns `gridAdapterState` from `cold` to `live`, and since #926 Accounts re-reads both registry facts. The old rows are replaced.
- **Unreachable target, or one whose first read is still in flight:** Accounts and the cockpit's agent-detail configuration tab keep showing instance A's agents and tenants under instance B's `boundProfileId`. The per-agent save statuses stay too.

## The Architectural Reality

Design authority: `TargetBinding`'s module summary.
> "rows a store holds belong to the profile whose answer produced them … A read through a bridge bound to ANOTHER profile first retires the previous profile's truth … so the new profile's first live answer is a first admission, and its failure shows its own cold truth instead of another instance's residents"

The two registry Stores are the surfaces still outside that rule.

- `apps/agentos/util/TargetBinding.mjs:42-71` (`retireRoster`) and `apps/agentos/view/ViewportController.mjs:333-362` (`switchToProfile`).
- `apps/agentos/view/accounts/Panel.mjs` already reads an unloaded definitions Store as "the Fleet has not answered yet" (#926). It clears its save statuses only when the Store instance is replaced.
- `apps/agentos/store/AgentDefinitions.mjs:8-10` still says "an empty store means no agent is defined yet, and the Accounts view says so". #926 made that untrue: an empty store that has not loaded is a registry that has not answered.

## The Fix

1. Retire both registry Stores with the rest of the old profile's truth: no rows, and the definitions Store reads as not loaded. Accounts then shows "the Fleet has not answered yet" until the new instance answers, and its save statuses for the old instance clear. Where the retirement lives (`TargetBinding` or the switch) is settled at intake against the existing retire seam.
2. `AgentDefinitions.mjs`'s summary says what an empty store means after #926: answered and empty, or not answered yet.

## Acceptance Criteria

- [ ] After a switch to an unreachable instance, Accounts lists no agent from the previous instance, says the Fleet has not answered, and the configuration tab holds no previous-instance definition. Unit spec, red on the base.
- [ ] After a switch to a reachable instance, Accounts lists that instance's agents once its first roster lands. Same spec.
- [ ] `AgentDefinitions.mjs`'s summary distinguishes an answered empty registry from an unanswered one.

## Out of Scope

- The tenants an instance can reach beyond what `listTenants` answers.
- The switcher's own UI.

## Related

#477 (row 2, parent: a switch to an unreachable instance is one of its provocations) · #925 / #926 · #181 (an instance switch binds its target)

Sweeps, 2026-11-12 03:3xZ:
- **Live latest-open + closed:** the latest 8 open (61 open, red band; #929..#919). All-state search: `instance switch agentDefinitions OR "previous instance" Accounts OR retireRoster tenants`. No equivalent; folds under #477.
- **A2A:** the source is Maryam's defect-note (`256048c4`); no claim on the surface.
- **MC:** `after switching to another Agent OS instance Accounts still shows the previous instance's agents and tenants`, 5 results, no prior decision.
- **Own-assignment:** 5 open (#925, #635, #633, #490, #414). #925 is the predecessor, nothing overlaps.

Retrieval Hint: "instance switch retire agentDefinitions fleetTenants TargetBinding unreachable profile Accounts"

🖖 Grace (Claude Opus 5.5, Claude Code)

## Timeline

- 2026-11-12T03:28:33Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-12T03:28:34Z @neo-opus-grace added the `bug` label
- 2026-11-12T03:28:34Z @neo-opus-grace added the `agent-os` label
- 2026-11-12T03:28:34Z @neo-opus-grace added the `ai` label
- 2026-11-12T03:28:38Z @neo-opus-grace added parent issue #477
- 2026-11-12T03:41:33Z @neo-opus-grace cross-referenced by PR #931
- 2026-11-12T04:15:29Z @tobiu referenced in commit `72dcd27` - "fix(agentos): a profile switch retires the Accounts registry rows (#930) (#931)

* fix(agentos): a profile switch retires the Accounts registry rows (#930)

Rows a Store holds belong to the profile whose answer produced them, which
is TargetBinding's rule. A switch to an instance that does not answer left
the previous instance's agents and tenants in Accounts and in the cockpit's
configuration tab. The controller now stamps the profile a registry read
lands under and fences a late answer from a replaced bridge. When the bound
profile moves on, the panel retires both Stores, with the definitions
unanswered, and the old profile's save statuses.

* test(agentos): re-stamp the visual baselines over the #930 inputs (#930)

AgentDefinitions.mjs, Controller.mjs and Panel.mjs are style-owning inputs
of the Darwin goldens. The goldens were re-verified on macOS at this head
with no golden changed, so only the three input entries move."
- 2026-11-12T04:15:30Z @tobiu closed this issue

