---
id: 1612
title: Three orchestrator modules build the tenant-repo key by hand
state: OPEN
labels:
  - enhancement
  - good first issue
  - ai
  - refactoring
  - agent-os
assignees: []
createdAt: '2026-11-11T19:48:03Z'
updatedAt: '2026-11-11T19:48:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1612'
author: neo-opus-vega
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
---
# Three orchestrator modules build the tenant-repo key by hand

## Context

A review follow-up from PR #1611 (#1608), named in Maryam's review 5864427019. #1608 was a defect of exactly this kind: two code paths built the key of the persisted tenant-repo manifest differently, so the operator's clear path found nothing and still reported success.

## The Problem

The key of one tenant repository, `<tenantId>/<repoSlug>`, is built in three modules under `ai/daemons/orchestrator/services/`. Read at `dev` `e49cd09d`:

- `TenantRepoSyncService.mjs`: `createTenantRepoAccessKey(repo)` (~:310, module-private). It writes the manifest; after PR #1611 its sweep and its clear both call this function.
- `DeploymentStateBridgeService.mjs`: `createTenantRepoLabel(repo)` (~:3909, module-private), the same body under another name. It reads the manifest with it: `persistedRevisions[createTenantRepoLabel(repo)]` at ~:2737 and ~:3588.
- `MaintenanceBackpressureService.mjs`: the template is inline in `resolveConfiguredTenantRepoLabels()` (~:74), and the labels index the manifest at ~:950.

All three produce the same string today. Nothing keeps them equal: a change to one leaves the others reading keys that no longer exist, and each module's specs stay green on hand-written fixtures. That is how #1608 stayed hidden through two pull requests.

## The Architectural Reality

- One writer, two readers, one persisted file (`tenant-repo-sync-revisions.json`).
- All three modules already import from `ai/daemons/orchestrator/scheduling/tenantRepoSync.mjs`, which holds the pure scheduling helpers (`isRepoDue`, `resolveUnknownRepoSelectorFailure`) and imports no service.
- Design authority: n/a. Nothing existing is called wrong; this removes two copies.

## The Fix

Export one function that builds the key and call it from all three modules. `scheduling/tenantRepoSync.mjs` is the suggested home, because every caller imports it already and no service has to import another one for it. `createTenantRepoAccessKey` and `createTenantRepoLabel` either go away or become that import under their local name.

It lands after PR #1611, which changes the same lines in `TenantRepoSyncService.mjs`.

## Acceptance Criteria

- [ ] One exported function builds the key. `TenantRepoSyncService`, `DeploymentStateBridgeService` and `MaintenanceBackpressureService` call it for every manifest read and write.
- [ ] No other copy of the template remains under `ai/daemons/orchestrator/`, except the two display strings and the jitter seed named under Out of Scope.
- [ ] A unit arm pins the produced string for a sample repo.
- [ ] The three modules' existing specs pass unchanged.

## Out of Scope

- Two display strings in `TenantRepoSyncService.mjs` (a WARN line at ~:1941 and a cadence label at ~:1961).
- The jitter seed in `computeDeterministicJitter` (`scheduling/tenantRepoSync.mjs` ~:308). It is a hash input, not a manifest key; changing it would move every repo's jitter.
- The key's format.

## Avoided Traps

- **A source-scan guard alone.** It pins the spelling of a template; it does not make the readers and the writer share one function.

## Related

#1608 · #1611

unowned-rationale: Ready for a contributor; maintainers provide planning and review. A refactor with no behaviour change and one new unit arm.

Live latest-open + closed sweep: latest 20 open of 151 (red band; no open ticket owns the key helper) and closed search (`tenant repo label key helper createTenantRepoLabel`), plus an organization search for both function names, at 2026-11-11T19:46Z; no equivalent found. A2A in-flight claim sweep: latest 30 messages, all read states, at 19:46Z; no claim on this surface. MC sweep: `query_raw_memories` refused twice at 19:46Z and 19:47Z (its embedding probe timed out while a tenant sync was embedding); in its place `ask_knowledge_base(type: 'ticket')` on "tenant repo key built separately in three services, shared helper for the manifest key": 5 references read, no prior decision. Own-assignment sweep: 16 open, none overlapping (#1608 is the parent of this follow-up).

Origin Session ID: 20e7f7fd-c32d-4575-92d1-fd9020bf5c31

Retrieval Hint: "createTenantRepoLabel createTenantRepoAccessKey manifest key three modules shared helper"

— Vega (Claude Fable 5.2, Claude Code) 🌿

## Timeline

- 2026-11-11T19:48:04Z @neo-opus-vega added the `enhancement` label
- 2026-11-11T19:48:05Z @neo-opus-vega added the `good first issue` label
- 2026-11-11T19:48:05Z @neo-opus-vega added the `ai` label
- 2026-11-11T19:48:05Z @neo-opus-vega added the `refactoring` label
- 2026-11-11T19:48:05Z @neo-opus-vega added the `agent-os` label
- 2026-11-11T19:48:16Z @neo-opus-vega cross-referenced by PR #1611
- 2026-11-11T20:49:36Z @neo-opus-vega cross-referenced by #1608

