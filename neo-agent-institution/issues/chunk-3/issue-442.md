---
id: 442
title: Carry the new Brain pin with forge-correct MCP controls
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - build
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-02T13:08:30Z'
updatedAt: '2026-10-02T16:17:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/442'
author: neo-gpt-emmy
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
blocking:
  - '[x] 448 The add-agent form defines a GitLab seat on its own instance'
  - '[x] 418 The roster card''s lane line and the detail''s lane pane read the roster row''s lane stamp'
closedAt: '2026-10-02T16:17:56Z'
---
# Carry the new Brain pin with forge-correct MCP controls

## Context

The next source pin can carry six merged Brain changes through `a9dd22ff3dd7c01ba8beb721402f6ec1a4fc874f`: neomjs/neo-agent-brain#742, #743, #745, #747, #748 and #749. It unblocks #418's lane-stamp consumer and supplies the hosted-preset/readiness and GitLab launch contracts. Grace offered this pin lane to Emmy; Ada confirmed the existing MCP-card alignment is outside her lanes.

## The Problem

A manifest-only bump would pair forge-aware backend defaults with GitHub-only controls. At the proposed Brain pin, a GitLab seat with `mcpServers:null` resolves GitLab workflow on and GitHub workflow off. `AgentConfigComponent` currently resolves and normalizes against the default catalog, displaying the reverse.

Executed pure-contract falsifier: start with GitLab `{gitlab-workflow:false}`, toggle Neural Link off using the current card's resolve/normalize chain, and the wire becomes only `{neural-link:false}`. The new backend then resolves GitLab workflow **on**. An unrelated edit discarded an explicit disable. The null/default case and unchanged GitHub case were separately checked. No live credential or provider operation was used.

## The Architectural Reality

`package.json`, `package-lock.json` and CI's explicit Brain checkout select the dependency. The range from `f9ccc2e` is ahead six/behind zero with no package dependency changes; its only shared `src/**` contract change is `src/fleet/contract/mcpServers.mjs`.

Brain's `mcpCatalogFor(forge)` owns the defaults. Its registry now normalizes against that catalog. Institution's `AgentDefinition` omits `forge`, while `AgentConfigComponent.onCardClick` and `createCardContent` call `resolveMcpMatrix` / `normalizeMcpOverrides` without it. These are the existing model and view boundaries to align; no second catalog belongs in Institution.

## The Fix

Advance the Brain manifest/lock/CI ref to the exact target above, keeping Engine `93769448934166a8c98b4d99eccda4c3d347caeb`. Carry the public forge in `AgentDefinition`; resolve, render and normalize the config card against the same canonical per-forge catalog, preserving legacy GitHub behavior. Cover the loss-of-disable counterexample and backend readback in the existing config/card tests. Refresh the visual stamp only after the relevant checks.

## Contract Ledger

| Surface | Authority | Behavior | Boundary | Evidence |
|---|---|---|---|---|
| Brain dependency and CI | manifests and explicit checkout | all select `a9dd22ff3dd7c01ba8beb721402f6ec1a4fc874f` | Engine unchanged | lock install and bound contract |
| `AgentDefinition.forge` | public Brain definition | retains the forge through hydration/readback | no credential or forge-changing intent | model/store control |
| MCP rows and sparse intent | Brain `mcpCatalogFor`, `resolveMcpMatrix`, `normalizeMcpOverrides` | display and save use the seat's same catalog | legacy GitHub behavior retained; no duplicated defaults | GitHub/GitLab defaults, explicit disable and unrelated toggle/readback controls |

Decision Record impact: aligned with existing public contract ownership; no new configuration or credential authority. Brain structure-map ran successfully; existing model/card/spec locations, no new module.

## Acceptance Criteria

- [ ] AC-1 Manifest, lock and CI carry the exact Brain target; the lock installs, and the Engine pin is unchanged.
- [ ] AC-2 GitHub/legacy and GitLab definitions retain their forge and show the backend's declared MCP states. A config edit and readback preserve unrelated explicit overrides, including GitLab workflow off.
- [ ] AC-3 Existing isolated and Brain-bound checks, relevant Accounts browser/component checks and Darwin visuals pass; the visual input stamp agrees.

## Out of Scope

The GitLab creation/credential form, #418's lane display, the #384/#440 wizard, Brain #750 orchestration, new Engine changes, real GitLab credentials/provider calls, and installation. The frozen #12 candidate remains untouched; real GitLab installed acceptance remains neomjs/neo-agent-brain#684's.

## Avoided Traps

Blindly bumping an additive contract; resolving with one catalog and normalizing with another; storing a fully resolved matrix; silently consuming a previously disabled workflow server; conflating source availability with installed delivery.

## Related

#418 · #414 · #351 · #12 · neomjs/neo-agent-brain#684 · neomjs/neo-agent-brain#729 · neomjs/neo-agent-brain#749.

## Sweeps

Latest 20 open Institution issues and 30 all-read-state A2A messages checked immediately before filing on 2026-10-02; no overlapping pin/card leaf. Own-assignment: #438 only, read and unrelated. Exact forge/MCP issue search returned no match. MC recovered #684's forge-credential context (memory `e2177b7d-6ac9-457b-99db-84fa1bd3f5b1`); other focused queries did not recover this consumer alignment. Live source, the executed controls and Ada's ownership check determine this scope.

Origin Session ID: 3acb1755-5285-4f3a-a74a-dae637bb629d
Retrieval Hint: Brain a9dd22f pin GitLab sparse overrides AgentDefinition forge AgentConfigComponent.

🪡 Emmy

## Timeline

- 2026-10-02T13:08:30Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-02T13:08:31Z @neo-gpt-emmy added the `enhancement` label
- 2026-10-02T13:08:31Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-02T13:08:32Z @neo-gpt-emmy added the `ai` label
- 2026-10-02T13:08:32Z @neo-gpt-emmy added the `build` label
- 2026-10-02T13:26:01Z @neo-opus-grace cross-referenced by PR #444
- 2026-10-02T13:30:04Z @neo-gpt-emmy marked this issue as blocking #418
- 2026-10-02T13:34:04Z @neo-gpt-emmy cross-referenced by PR #445
- 2026-10-02T14:02:41Z @neo-opus-grace cross-referenced by #448
- 2026-10-02T14:02:56Z @neo-opus-grace marked this issue as blocking #448
- 2026-10-02T16:17:56Z @tobiu referenced in commit `b5c26cd` - "feat(deps): align Fleet with forge-aware Brain contracts (#442) (#445)"
- 2026-10-02T16:17:56Z @tobiu closed this issue
- 2026-10-02T16:27:35Z @neo-gpt-emmy cross-referenced by #451
- 2026-10-02T17:19:17Z @neo-gpt-sophie cross-referenced by PR #450

