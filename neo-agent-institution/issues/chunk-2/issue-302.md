---
id: 302
title: 'Brain pin to dev, with the harness resolving fleet.agentsRoot'
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-27T15:23:11Z'
updatedAt: '2026-09-28T10:03:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/302'
author: neo-opus-ada
commentsCount: 2
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
closedAt: '2026-09-28T10:03:08Z'
---
# Brain pin to dev, with the harness resolving fleet.agentsRoot

## Context

The FM reaches today's Brain work only through the Brain pin (`package.json`, `c6c92c2`). Dev now carries the plane's deployment snapshot for the System view (neomjs/neo-agent-brain#582), the PAT-required registry (neomjs/neo-agent-brain#577), the agents root (neomjs/neo-agent-brain#573) and the LaunchAgent PATH (neomjs/neo-agent-brain#575). The bump is blocked. `harness/brain.mjs` resolves `AiConfig.fleet.instanceRoot` and sets `NEO_FLEET_INSTANCE_ROOT`, and neomjs/neo-agent-brain#573 retired both for `fleet.agentsRoot` / `NEO_FLEET_AGENTS_ROOT`. So the harness contract run goes red on the bumped Brain (Grace, 14:59Z).

## The Fix

One change, because neither half works alone:
- Pin the Brain to its current dev head.
- The harness resolves `AiConfig.fleet.agentsRoot` as `fleetAgentsRoot` and places it through `NEO_FLEET_AGENTS_ROOT`: under the smoke isolation root, and under the per-user data root when packaged.

## Acceptance Criteria

- [ ] AC-1 The Brain pin is the current dev head, and the lock agrees.
- [ ] AC-2 `harness/brain.mjs` and `brain.spec` name only `agentsRoot` / `NEO_FLEET_AGENTS_ROOT`, and the harness contract run is green on the bumped Brain.
- [ ] AC-3 Unit, components and E2E pass on the bumped Brain.

## Post-Merge Validation

- The installed FM's System view shows the plane's snapshot after repackage (neomjs/neo-agent-brain#582's app-visible proof).

Related: #301 (the engine pin; adjacent lines, so whichever lands second rebases) · #7 · #10

Live latest-open sweep (2026-09-27T15:25Z): the open issues and PRs of this repository. #300 / #301 pin the engine only; no Brain pin ticket. Grace's 14:59Z broadcast hands the harness lines to me.

Authored by Ada (Claude Opus 5.5, Claude Code). Session f3d50317-fe3b-4773-b4ac-db05e1fa6812.


## Timeline

- 2026-09-27T15:23:13Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-27T15:23:13Z @neo-opus-ada added the `enhancement` label
- 2026-09-27T15:23:13Z @neo-opus-ada added the `ai` label
- 2026-09-27T15:28:03Z @neo-opus-ada cross-referenced by PR #303
- 2026-09-27T15:35:01Z @neo-opus-ada referenced in commit `4b6350f` - "chore(deps): Brain pin to dev 03b07cc, with the harness resolving fleet.agentsRoot (#302)

The pin carries the plane's deployment snapshot (neo-agent-brain#582), the
PAT-required registry (#577), the agents root (#573) and the LaunchAgent PATH
(#575). #573 retired fleet.instanceRoot and NEO_FLEET_INSTANCE_ROOT, so the
harness resolves fleet.agentsRoot and places it through NEO_FLEET_AGENTS_ROOT in
the same change; the CI contract job checks out the same Brain commit."
- 2026-09-27T15:38:51Z @neo-opus-ada referenced in commit `e52a23d` - "chore(deps): Brain pin to dev d5cd907, with the harness resolving fleet.agentsRoot (#302)

The pin carries the Golden Path's complete recommendation (neo-agent-brain#586),
the plane's deployment snapshot (#582), the PAT-required registry (#577), the
agents root (#573) and the LaunchAgent PATH (#575). #573 retired
fleet.instanceRoot and NEO_FLEET_INSTANCE_ROOT, so the harness resolves
fleet.agentsRoot and places it through NEO_FLEET_AGENTS_ROOT in the same change;
the CI contract job checks out the same Brain commit."
- 2026-09-27T15:46:58Z @tobiu referenced in commit `99c48f6` - "Merge pull request #303 from neomjs/ada/302-brain-pin

chore(deps): Brain pin to dev d5cd907, with the harness resolving fleet.agentsRoot (#302)"
- 2026-09-27T15:46:59Z @tobiu closed this issue
- 2026-09-28T08:19:21Z @neo-gpt-emmy reopened this issue
- 2026-09-28T08:19:24Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-28T08:19:24Z @neo-gpt-emmy unassigned from @neo-opus-ada
### @neo-gpt-emmy - 2026-09-28T08:19:25Z

**`[lane-override]` reassignment audit-trail** (#11537 §AC8)

**Previous assignees:** `@neo-opus-ada`
**New assignees:** `neo-gpt-emmy`
**Reason:** Ada explicitly handed the final Brain pin to Emmy in MESSAGE:51cbedd0-ebc0-4850-9b1c-4971d2880b3b on 2026-09-27. The initial #303 harness rename is merged, but current dev and installed FM still pin d5cd907, before the merged whole-graph reader #587. Tobi's 2026-09-28 priority is Graph experience first; reopening this existing pin ticket for that remaining delivery.

*Audit-trail per AGENTS.md §6.5 — `acknowledgedReassign` reason persistence. Graph-ingested via Retrospective daemon comment-scan path.*

### @neo-gpt-emmy - 2026-09-28T08:21:38Z

Reopened for the remaining Brain pin delivery under Ada's explicit handoff to Emmy (Memory Core `MESSAGE:51cbedd0-ebc0-4850-9b1c-4971d2880b3b`, 2026-09-27), after #303's harness rename and #304's reader landed. The harness rename remains delivered.

Live verification on 2026-09-28: Institution `dev` still pins Brain `d5cd907`; the installed bundle's receipt carries that same revision. Brain #587 and #592 are merged. The installed Observatory shows 18 nodes / 20 edges, while a current `get_graph_scene` read returns 149,541 nodes / 216,775 edges / 69,813 unlinked nodes, with neither node nor edge truncation. Tobi has chosen the graph experience as today's first visible milestone.

Prescription checked: `package.json`, `package-lock.json` and `.github/workflows/ci.yml` own this dependency/contract alignment. The remaining change is their Brain revision, followed by package and installed verification under #10. No new graph renderer or harness path change is needed for the pin. The live Institution PR queue was empty at intake. The header and Team lens design remain under #10 / neomjs/neo#19151 and are not certified by this dependency update.

Emmy · session 01a0e711-293d-7d43-b01a-3a88bb68dbf6

- 2026-09-28T09:49:08Z @neo-gpt-emmy cross-referenced by PR #307
- 2026-09-28T10:03:08Z @tobiu referenced in commit `f710502` - "Merge pull request #307 from neomjs/codex/302-fleet-graph-pin

feat(fleet): consume the full-graph Brain revision (#302)"
- 2026-09-28T10:03:08Z @tobiu closed this issue
- 2026-09-28T10:09:52Z @neo-gpt-emmy cross-referenced by #308
- 2026-09-28T10:09:54Z @neo-gpt-emmy cross-referenced by #309

