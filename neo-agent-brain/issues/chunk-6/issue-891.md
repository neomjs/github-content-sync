---
id: 891
title: 'participation.mjs cannot write an existing identity: the command never loads Neo''s instance manager'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-05T15:15:01Z'
updatedAt: '2026-10-05T17:33:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/891'
author: neo-opus-vega
commentsCount: 0
parentIssue: 28
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-05T17:33:33Z'
---
# participation.mjs cannot write an existing identity: the command never loads Neo's instance manager

## Context

#883's post-merge receipt, run on the plane at Brain `1879b588` (2026-10-05, 15:13Z), in the Memory Core container:

```
node ai/scripts/fleet/participation.mjs bench --identity @agent-a --reason "…" --apply
TypeError: Neo.get is not a function
    at RecordFactory.onRecordChange (neo.mjs/src/data/RecordFactory.mjs:375)
    at GraphService.upsertNode (ai/services/memory-core/GraphService.mjs:387)
    at recordParticipation (ai/services/memory-core/recordParticipation.mjs:87)
```

The command exits 1 and writes nothing. `show` and the dry run work, because neither updates a record.

## The Problem

Updating an existing node calls `Record.set`, which notifies the record's store through `Neo.get`. Neo's instance manager defines `Neo.get`, and the command imports only `Neo.mjs` and `core/_export.mjs` before loading `GraphService`. `services.mjs` and `recreateGraphDb.mjs` import `neo.mjs/src/manager/Instance.mjs` beside them. #883's specs drove the command over a GraphService double, which never reaches `RecordFactory`.

## The Fix

1. Import `neo.mjs/src/manager/Instance.mjs` in `ai/scripts/fleet/participation.mjs`, as `recreateGraphDb.mjs` does.
2. A spec that runs the real entrypoint in fresh processes against a temp graph (`NEO_MEMORY_DB_PATH`): seed with `seedAgentIdentities.mjs`, then `bench --apply`, `show`, the seeder again, and `show`. That reproduces this failure, and it also proves #883's reseed precedence on a seeded identity end to end.

## Acceptance Criteria

- [ ] The entrypoint benches and returns an existing identity in a fresh process (spec over a temp graph; red on `dev`).
- [ ] In the same spec, a reseed after the bench keeps the decision and logs "participation kept".
- [ ] Post-merge, after a plane cut carrying this, #883's receipt runs on the plane: `@agent-a` benched, then returned, read through `who_is_online` both times. Receipt on #883.
  Residual-Owner: #28

## Out of Scope

The command's surface and semantics (#883).

## Related

#883 · #884 · #28

Live latest-open sweep: latest 12 open issues at 2026-10-05T15:14Z, no equivalent; "Neo.get is not a function" search empty.

Origin Session ID: 79265a5a-6888-4d34-94ee-0d933cbacff1

## Timeline

- 2026-10-05T15:15:02Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-05T15:15:09Z @neo-opus-vega added parent issue #28
- 2026-10-05T15:16:15Z @neo-opus-vega added the `bug` label
- 2026-10-05T15:16:15Z @neo-opus-vega added the `ai` label
- 2026-10-05T15:16:16Z @neo-opus-vega added the `agent-os` label
- 2026-10-05T15:17:01Z @neo-opus-vega cross-referenced by PR #892
- 2026-10-05T15:17:52Z @neo-opus-vega cross-referenced by #883
- 2026-10-05T17:33:33Z @tobiu referenced in commit `b525a12` - "fix(fleet): participation.mjs loads Neo's instance manager, so it can write an existing identity (#891) (#892)"
- 2026-10-05T17:33:33Z @tobiu closed this issue

