---
id: 1411
title: '--clear-backoff refuses every repo that kb-config.yaml configures'
state: CLOSED
labels:
  - bug
  - good first issue
  - ai
assignees:
  - neo-gemini-julia
createdAt: '2026-10-31T21:27:22Z'
updatedAt: '2026-11-02T04:04:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1411'
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
closedAt: '2026-11-02T04:04:34Z'
---
# --clear-backoff refuses every repo that kb-config.yaml configures

## Context

On 2026-10-31 the local plane was cut to Brain dev `63e99c65`, which carries the KB embed fix from #1405. `neo-shared/github-content-sync` was still suppressed by the backoff it had built up during the outage (`consecutiveFailures=3, backoffX=8`, next attempt ~23:15Z). The CLI documents `--clear-backoff` for exactly this case: "after repairing a shared dependency, when the streak recorded during the outage would otherwise suppress a lane that now works". On this plane it refused:

```
$ docker exec neo-local-agent-os-orchestrator-1 sh -c 'cd /app && node ./ai/scripts/maintenance/syncTenantRepos.mjs --clear-backoff --repo-slug github-content-sync'
[WARN] [TenantRepoSync] clear-backoff: requested repoSlug(s) not configured: github-content-sync. Configured: (none).
{"status": "failed", "details": {"reason": "repo-not-configured", "reasonCode": "KB_TENANT_REPO_SYNC_REPO_NOT_CONFIGURED", "unknownSlugs": ["github-content-sync"], "configuredSlugs": []}}
```

On every cycle, the sweep in the same container logs `neo-shared/github-content-sync` and its siblings `create-app`, `devindex`, `devindex-opt-in` and `devindex-opt-out`.

## The Problem

`clearTenantRepoBackoff` resolves its repository list as `tenantReposConfig || AiConfig.data.orchestrator.tenantRepoSync`, which is only the last of three tiers. The sweep resolves the same list through `resolveTenantReposConfig()`, which calls `KnowledgeBaseIngestionService.listConfiguredTenantRepos`: a graph node first, then the `kb-config.yaml` bootstrap, then `aiConfig.tenantRepos[]`. On a plane whose repositories come from `kb-config.yaml` or a graph node, the AiConfig tier is empty. Every selector is then reported unknown, and an unscoped clear reports "no tenantRepos configured; nothing to clear".

This is the blind spot that neo `#14404` (graph-only tenantRepo config is invisible to pull-mode sync) closed for the sweep. The resolver's JSDoc says it "replaces the prior direct `aiConfig.tenantRepos` read so the documented bootstrap / graph tiers are actually honored". The clear arrived later, in neo `#17067` / PR `#17352`, and read AiConfig directly.

Nothing caught it because every spec injects `tenantReposConfig` (`TenantRepoSyncService.spec.mjs` ~:328, ~:389, ~:429), and the CLI never passes it (`syncTenantRepos.mjs` ~:206). The production branch therefore has no coverage.

## The Architectural Reality

- `ai/daemons/orchestrator/services/TenantRepoSyncService.mjs`: `clearTenantRepoBackoff` (~:4061) and `resolveTenantReposConfig` (~:3686). Three consumers already use the resolver: the sweep (`runTask`, ~:1918), `DeploymentStateBridgeService.mjs` ~:2498 and `MaintenanceBackpressureService.mjs` ~:70.
- `ai/scripts/maintenance/syncTenantRepos.mjs`: the CLI. The clear already runs under the heavy-maintenance lease, so an async resolution changes nothing about its exclusion.
- Structure map: owned by `ai/daemons/orchestrator/services`, and no new file is needed.

## The Fix

In `clearTenantRepoBackoff`, resolve the list the way the sweep does:

```js
const resolvedConfig = tenantReposConfig || await this.resolveTenantReposConfig();
```

The injected `tenantReposConfig` seam stays for tests. The new unit arm must not write to `AiConfig` (ADR 0019 §4, B4). Stub `resolveTenantReposConfig` on the service inside a `try/finally`, or give the clear the same `ingestionService` seam that `resolveTenantReposConfig({ingestionService})` already accepts.

## Acceptance Criteria

- [ ] A unit arm calls `clearTenantRepoBackoff` without `tenantReposConfig`, while the resolver yields a repository the AiConfig tier does not carry, and the streak clears. Red on `dev` first.
- [ ] The existing clear-backoff arms stay green unchanged.
- [ ] (post-merge, after a plane cut carrying it) On the local plane, `syncTenantRepos.mjs --clear-backoff --repo-slug <a configured slug>` reports `completed` or `already-clear`, never `repo-not-configured`. A maintainer posts the receipt here.

## Out of Scope

- The backoff and recovery semantics (#1382 / PR #1404).
- The embed failure that built the streak (#1405 / PR #1406).

## Decision Record impact

Aligned with ADR 0019 (`0019-aiconfig-reactive-provider-ssot.md`). No leaf, alias, env read or config write is added. A direct subtree read that skipped the documented tiers gives way to the resolver every other consumer already uses.

## Related

- Parent area: #64 (tenant ingestion).
- Found while verifying #1405's post-merge AC on the plane.

Live latest-open sweep: the latest 20 open Brain issues, read at 2026-10-31T21:26:11Z; no equivalent. `gh search issues "clear-backoff"` finds no ticket on this defect.
A2A sweep: the last 30 messages, all read states, at 21:26Z; no claim on clear-backoff or the tenant repo resolver.
MC sweep: "clear-backoff refuses a configured tenant repo as not configured; kb-config.yaml tenant repos invisible to the backoff reset", 6 results. The neighbour is Euclid's neo `#14404` (07-01), which fixed the same blind spot on the sweep path.
Own-assignment sweep: my open Brain tickets on this surface are #64 (the parent area) and #1403 (the embed trace); neither covers the clear.

unowned-rationale: Ready for a contributor; maintainers provide planning and review. The fix is one line plus a unit arm, and the post-merge plane receipt is a maintainer's.

Retrieval Hint: "clear-backoff Configured: (none) kb-config.yaml tenantRepos resolver"


## Timeline

- 2026-10-31T21:27:23Z @neo-opus-vega added the `bug` label
- 2026-10-31T21:27:23Z @neo-opus-vega added the `good first issue` label
- 2026-10-31T21:27:24Z @neo-opus-vega added the `ai` label
- 2026-10-31T21:30:57Z @neo-opus-vega cross-referenced by PR #1414
- 2026-10-31T21:33:08Z @neo-opus-vega cross-referenced by #64
- 2026-11-02T03:39:27Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-02T03:42:09Z @neo-gemini-julia cross-referenced by PR #1427
- 2026-11-02T04:04:34Z @tobiu referenced in commit `38e7d91` - "fix(orchestrator): resolve tenantReposConfig across all tiers in clearTenantRepoBackoff (#1411) (#1427)"
- 2026-11-02T04:04:36Z @tobiu closed this issue

