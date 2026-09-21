---
id: 197
title: Establish deploy/host and independent deploy/cloud packages
state: CLOSED
labels:
  - enhancement
  - ai
  - refactoring
  - architecture
  - build
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-08-27T15:06:38Z'
updatedAt: '2026-08-28T22:22:12Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/197'
author: neo-gpt-emmy
commentsCount: 1
parentIssue: 192
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 202 Replace Engine-era learning folders with one Brain journey'
  - '[x] 200 Unify embedding admission across provider paths'
  - '[x] 199 Move Dream into a Cloud-owned domain'
closedAt: '2026-08-28T13:00:51Z'
---
# Establish deploy/host and independent deploy/cloud packages

## Context

Brain #192 owns the real package boundary under parent Epic #189. Current deployment files mix three Host-Edge definitions and fifteen Cloud definitions under `ai/deploy/**`; root `package.json` exposes both planes.

## The Problem

The repository does not enforce the settled plane topology. Cloud operations remain runnable from the Host root, no independent Cloud install exists, and path placement does not tell an operator which plane owns a definition.

## The Architectural Reality

Operator authority is exact: `deploy/host/**` for Host Edge; `deploy/cloud/**` for Container Cloud; root manifest contains Host-Edge scripts only; `deploy/cloud/package.json` is an independently installed nested package; no npm workspaces or ancestor-hoist dependency.

Pre-Flight (structural full): considered legacy `ai/deploy`, root `deploy`, and nested `cloud/deploy`; the accepted Host/Cloud boundary and current operator correction select `deploy/host` plus `deploy/cloud`. Scalability keeps each plane self-contained; ADR 0040's no-workspaces invariant binds; the nested Cloud package reduces future extraction friction. Map maintenance is required because these are new canonical homes.

Scope boundary from live closure evidence: this leaf isolates deployment definitions and command surfaces. The current `ai:host-edge` entrypoint still reaches `chromadb` and `better-sqlite3` through the legacy mixed Orchestrator closure. Removing that source-level reach belongs to the parent source/domain refactor; this leaf must not claim dependency isolation by moving manifest rows alone.

## The Fix

Move all eighteen deployment definitions into the two settled directories. Create the independent Cloud manifest and lockfile with Cloud-owned scripts and explicit dependencies. Remove Cloud scripts from the root manifest; retain only proven Host operations there. Rewrite build contexts, non-dev binds, and references to the new paths; retain exactly the three writable dev-profile repo binds that make live iteration live. Delete `ai/deploy/**`.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| root package | Host-Edge contract | Host operations only | missing command fails loud | Host runbook | root script/readback |
| `deploy/cloud/package.json` | Cloud boundary | independent install and commands | no ancestor fallback | Cloud runbook | isolated install/build |
| deployment definitions | plane ownership | one path under Host or Cloud | no legacy alias | updated references | zero `ai/deploy` paths |

## Decision Record impact

Aligned with ADR 0040's plane separation and no-workspaces invariant; reconciles stale path vocabulary to the operator-set final topology.

## Acceptance Criteria

- [ ] Three Host definitions live under `deploy/host/**`; fifteen Cloud definitions live under `deploy/cloud/**`; `ai/deploy/**` is absent.
- [ ] Root `package.json` exposes Host-Edge operations only.
- [ ] `deploy/cloud/package.json` and lockfile install independently with no workspace or ancestor-hoist reliance.
- [ ] Every build context and non-dev bind resolves inside the Cloud package or explicit immutable dependencies; the dev profile retains exactly three writable live-repository binds for KB, MC, and Orchestrator iteration.
- [ ] The Host root cannot execute Cloud scripts or resolve the nested Cloud package through a workspace/ancestor hoist; remaining legacy source-level Cloud-driver reach is named as parent work, not claimed solved here.
- [ ] Current deployment guides reference only the final paths and package-owned commands.
- [ ] No new inventory, manifest, or topology-proof subsystem is added.

## Out of Scope

Changing deployment behavior; production cutover; Dream/embedding refactoring; Engine projection removal; refactoring the Host-Edge Orchestrator's mixed source closure.

## Avoided Traps

- `deploy/` plus `cloud/deploy/`.
- Forwarding Cloud commands from the Host root.
- Workspaces, hoisting, or compatibility aliases.

## Related

Parent: #192 · Goal: #189 · Related: #12, #90

Origin Session ID: d39e8182-295f-418a-82cd-a96be9c08e4f

Retrieval Hint: `deploy host deploy cloud nested package root Host Edge scripts only`


## Timeline

- 2026-08-27T15:06:40Z @neo-gpt-emmy added the `enhancement` label
- 2026-08-27T15:06:41Z @neo-gpt-emmy added the `ai` label
- 2026-08-27T15:06:41Z @neo-gpt-emmy added the `refactoring` label
- 2026-08-27T15:06:41Z @neo-gpt-emmy added the `architecture` label
- 2026-08-27T15:06:42Z @neo-gpt-emmy added the `build` label
- 2026-08-27T15:06:42Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-27T15:08:12Z @neo-gpt-emmy cross-referenced by #189
- 2026-08-28T08:04:27Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-08-28T09:27:21Z @neo-opus-vega unassigned from @neo-opus-vega
- 2026-08-28T10:14:56Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-08-28T10:58:29Z @tobiu referenced in commit `f09b0be` - "feat(deploy): establish host and cloud packages (#197)"
- 2026-08-28T11:00:18Z @tobiu cross-referenced by PR #205
- 2026-08-28T11:04:18Z @tobiu referenced in commit `255fb1b` - "feat(ci): include Cloud scripts in plane lint (#197)"
- 2026-08-28T11:12:28Z @tobiu referenced in commit `423df81` - "feat(ci): install Cloud package for integration suites (#197)"
- 2026-08-28T11:15:40Z @tobiu referenced in commit `e78af86` - "feat(ci): test exact Brain head in integration suites (#197)"
- 2026-08-28T11:25:19Z @tobiu referenced in commit `80a807d` - "feat(ci): preserve lock in exact-head integration artifact (#197)"
- 2026-08-28T13:00:51Z @tobiu referenced in commit `7516d9b` - "Merge pull request #205 from neomjs/codex/197-deploy-package-boundaries

feat(deploy): establish Host and Cloud packages (#197)"
- 2026-08-28T13:00:52Z @tobiu closed this issue
- 2026-08-28T16:49:21Z @neo-gpt-emmy cross-referenced by PR #208
- 2026-08-28T22:14:39Z @neo-gpt-emmy cross-referenced by #213
- 2026-08-28T22:16:16Z @neo-gpt-emmy cross-referenced by #214
- 2026-08-28T22:22:06Z @neo-gpt-emmy cross-referenced by #192
### @neo-gpt-emmy - 2026-08-28T22:22:12Z

Architecture correction: #213 supersedes the package/source prescription this closed ticket implemented. #214 now removes package authority and executable JavaScript from `deploy/**`; Host and Cloud remain profiles over one canonical source tree. Do not reuse #197 as placement precedent.

- 2026-08-29T19:47:50Z @neo-opus-vega cross-referenced by PR #236
- 2026-09-05T00:54:44Z @neo-fable-clio cross-referenced by #324
- 2026-09-21T10:38:48Z @neo-opus-grace cross-referenced by PR #397
- 2026-09-21T10:49:50Z @neo-opus-grace cross-referenced by #201

