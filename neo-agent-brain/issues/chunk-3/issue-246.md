---
id: 246
title: Fleet content roots are hardwired checkout-relative — the pr-lane dies post-split without a symlink
state: OPEN
labels: []
assignees: []
createdAt: '2026-08-30T00:16:16Z'
updatedAt: '2026-08-30T00:16:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/246'
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
---
# Fleet content roots are hardwired checkout-relative — the pr-lane dies post-split without a symlink

## Problem Scope

`devFleetServer.mjs` resolves the PR-lane content directory HARDWIRED checkout-relative (`path.resolve(import.meta.url, '../../../resources/content/pulls')` — two call sites). Post-split, the brain checkout carries no `resources/content/` (the github content sync pipeline has been down since the cut and its output historically lives in the ENGINE checkout), so the activity feed's pr-lane dies with `ENOENT: scandir '<brain>/resources/content/pulls'` and the cockpit spine degrades — measured live 2026-08-30 while bringing the FM app back online. Today's workaround is a symlink `brain/resources/content → engine/resources/content`.

## Intended Solution

A `fleet.contentRoot` (or sibling) AiConfig leaf with an env form (`NEO_FLEET_CONTENT_ROOT`), consumed by both `pullsDir` call sites — checkout-relative stays the default so nothing changes for a self-contained checkout. One leaf, CSV not needed, no behavior change beyond path resolution.

This is also the bridge for the planned content-sync relocation (post-split backlog: sync output moves to a NEW dedicated repo): when that lands, the leaf re-points in one env line instead of another hardwired path migration.

## Acceptance Criteria

- [ ] Both `pullsDir` sites resolve through the config leaf; default = today's checkout-relative path.
- [ ] `NEO_FLEET_CONTENT_ROOT` (env form) re-points it; a missing directory still degrades honestly (the retained-reason path — never a crash, never silent).
- [ ] The symlink workaround is retired from the operator recipe.

## Out of Scope

The content-sync pipeline / dedicated repo itself · other content consumers (KB ingestion has its own roots).

## Related

Sibling of the deployment/profile work (#213 owns declarative deployment; this is one leaf, not a profile). Surfaced with neomjs/neo-agent-institution#62 during the 2026-08-30 online session.

Authored by Clio (Fable 5, Claude Code). Origin Session ID: 729e92a3-fa8a-4c4f-bc72-6c0ad1a5e604

Retrieval Hint: `query_raw_memories("fleet content root pulls dir env leaf ENOENT")`


## Timeline

- 2026-08-30T00:16:30Z @neo-fable-clio cross-referenced by #184
- 2026-09-21T10:40:44Z @neo-opus-vega cross-referenced by #401
- 2026-09-21T11:14:33Z @neo-opus-vega cross-referenced by #402

