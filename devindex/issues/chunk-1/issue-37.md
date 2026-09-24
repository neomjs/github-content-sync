---
id: 37
title: 'The publisher uses Storage before its ready(), so a publish can race ensureFiles'
state: CLOSED
labels:
  - bug
assignees:
  - neo-opus-ada
createdAt: '2026-09-24T14:46:00Z'
updatedAt: '2026-09-24T15:22:43Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/37'
author: neo-opus-ada
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
closedAt: '2026-09-24T15:22:43Z'
---
# The publisher uses Storage before its ready(), so a publish can race ensureFiles

## Context

CI on neomjs/devindex#36 (`test`, 14:44Z) failed on a flaky arm that #36 does not touch. `WorkingSetRunBoundary.spec.mjs`'s "the collapse check compares with the run's start, not the public seed" failed once, then passed on retry, and `--fail-on-flaky-tests` turns that into a red job. The failing run's publisher said:

```
[publish] optin-sync.json is missing or empty — refusing to publish a set that would truncate the next run.
```

## The Problem

`buildScripts/publishWorkingSet.mjs` uses the `Storage` singleton without awaiting `Storage.ready()`. `Storage#initAsync` runs `ensureFiles()`, which creates each missing working-set member one `await` at a time, with `optinSync` last (`apps/devindex/services/Storage.mjs:53–80`). The publisher's `assertReadable` loop can reach `optin-sync.json` before `ensureFiles()` has written it. The Manager does it right: `await Storage.ready()` before hydrating (`Manager.mjs:138`).

In the workflow, the stages that run before the publish have already written every member, so production does not hit this today. The arm does, because its data directory holds only `users.jsonl` and the run mark. It is still the same defect: a Neo singleton used before its `ready()`.

## The Fix

`publish()` awaits `Storage.ready()` before touching the working set.

## Acceptance Criteria

- [ ] `publishWorkingSet.mjs` awaits `Storage.ready()` first.
- [ ] `WorkingSetRunBoundary.spec.mjs` runs on a slow disk: its preload makes every `fs/promises.access` wait 30 ms. The collapse arm then fails deterministically before the fix, with CI's exact message, and passes after it. *(Amended: `--repeat-each 20` passed 100/100 locally on dev, because a fast disk never loses the race.)*

## Related

#29 · #32 (where the arm came from) · neomjs/devindex#36 (the run that surfaced it)

Sweeps: the open issues here at 2026-09-24T14:45:47Z (#1, #9, #33). No equivalent.

Origin Session ID: 101d2ce9-9f43-4f5a-9ae2-3c75bf8f6fcf

Authored by ⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code



## Timeline

- 2026-09-24T14:46:01Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-24T14:46:01Z @neo-opus-ada added the `bug` label
- 2026-09-24T14:47:42Z @neo-opus-ada cross-referenced by PR #38
- 2026-09-24T15:22:43Z @tobiu referenced in commit `3d6fc3c` - "Merge pull request #38 from neomjs/ada/37-publisher-ready

fix(data-sync): the publisher waits for Storage's ready() before reading the set (#37)"
- 2026-09-24T15:22:43Z @tobiu closed this issue

