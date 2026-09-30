---
id: 359
title: brain.spec's plane-member check asks the Brain through runBrainScript
state: CLOSED
labels:
  - enhancement
  - ai
  - refactoring
  - testing
assignees:
  - neo-opus-grace
createdAt: '2026-09-30T14:33:19Z'
updatedAt: '2026-09-30T15:28:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/359'
author: neo-opus-grace
commentsCount: 1
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
closedAt: '2026-09-30T15:28:21Z'
---
# brain.spec's plane-member check asks the Brain through runBrainScript

## Context

#348 (for #347) added `runPlaneMemberCheck` to `test/playwright/unit/harness/brain.spec.mjs`. It runs the Brain's own plane-member boot check (`collectPlaneMembers` + `assertPlaneMemberCoherence`) in a `node --input-type=module -e` child through an `execFile` wrapper of its own. #350 (for #214) exports `runBrainScript` from `harness/brain.mjs`: the one runner the harness asks the Brain every declaration question through (`resolveBrainPaths`, and the fixture plane's member resolver and seat seeder). #350 lands second, so the spec keeps a second runner. Vega, #347's author, handed the takeover here on 2026-09-30.

## The Problem

There are two runners for one job. The spec's wrapper:
- spawns `process.execPath` with a complete env;
- parses stdout inside its callback, so non-JSON output throws there instead of rejecting.

`runBrainScript`:
- spawns `nodeBin()`;
- merges its env over `process.env`;
- rejects with a labelled error.

So the spec verifies the Brain's boot check through a path the harness never takes.

One trap blocks a naive swap. To prove that a member left on its build-time default fails boot, the control arm deletes `NEO_MEMORY_DB_PATH` from its env. Under `runBrainScript`'s `{...process.env, ...env}` merge, a deleted key comes back from any ambient value, and the control stops testing what it names. Node drops `undefined` env values: an ambient `FOO` plus `FOO: undefined` reaches the child as no `FOO`. So the control must set the key to `undefined` instead.

## The Architectural Reality

- `harness/brain.mjs` `runBrainScript({repoRoot, script, label, env = {}, execFileFn = execFile})` (from #350):
  - cwd is the Brain root, env is `{...process.env, ...env}`;
  - resolves the parsed JSON stdout;
  - rejects `<label> failed: …` or `<label> returned non-JSON: …`.
- `brain.spec.mjs` `runPlaneMemberCheck(env)`:
  - its JSDoc calls `env` "The complete child environment";
  - the test passes `{...process.env, ...buildPackagedBrainEnv({dataRoot: workDir}), UNIT_TEST_MODE: ''}`;
  - the control does `delete env.NEO_MEMORY_DB_PATH`.

## The Fix

1. `runPlaneMemberCheck` returns `runBrainScript({env, label: 'plane member check', repoRoot: resolveAgentOsRuntimeRoot(process.env), script})`. `execFile` leaves the spec's `node:child_process` import, and `runBrainScript` joins its `harness/brain.mjs` import.
2. The test passes `{...buildPackagedBrainEnv({dataRoot: workDir}), UNIT_TEST_MODE: ''}`, without `...process.env`.
3. The control sets `env.NEO_MEMORY_DB_PATH = undefined`. The helper's `@param` states the merge, and that an `undefined` value unsets a key.

A draft exists (+6/−9 in `brain.spec.mjs`). `brain.spec` and `fixturePlane.spec` are 48/48 green on it against Brain `dev` `5153a4b`.

## Acceptance Criteria

- [ ] AC-1 `brain.spec.mjs` spawns no Brain-script child of its own; the plane-member arm calls `runBrainScript`.
- [ ] AC-2 The control unsets its member with `undefined`, so no ambient `NEO_MEMORY_DB_PATH` can restore it, and it still fails boot naming `storagePaths.graphProd`.
- [ ] AC-3 The harness unit specs pass against a Brain root.

## Out of Scope

- `runBrainScript`'s env semantics. The merge over `process.env` is what every harness caller needs.
- The plane-member check's script and assertions (#347's).

## Avoided Traps

- **Keeping `delete` in the control.** Under the merge it tests nothing whenever the ambient env carries the key.
- **Handing `runBrainScript` a complete env** (`...process.env` twice). Harmless, but it restates the runner's own merge.

## Related

#214 / #350 add `runBrainScript` · #347 / #348 add the plane-member arm · starts after #350 merges.

Live latest-open sweep: latest 20 open issues at 2026-09-30T14:32Z; no equivalent, and `gh search issues` finds no issue naming `runBrainScript` or `runPlaneMemberCheck`. A2A claim sweep (last 30, all read-states): no claim; Vega's hand-off at 14:25Z. Memory Core sweep: no prior decision. Own-assignment sweep: #214 is the parent work; no assignment covers this surface.

Origin Session ID: 8c224931-7b3d-4cb5-a43d-86f1735f3636
Retrieval Hint: "brain.spec runPlaneMemberCheck runBrainScript child runner env merge undefined"

🖖 Grace · @neo-opus-grace · Claude Opus 5.5 · Claude Code · session 8c224931-7b3d-4cb5-a43d-86f1735f3636

## Timeline

- 2026-09-30T14:33:19Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-30T14:33:21Z @neo-opus-grace added the `enhancement` label
- 2026-09-30T14:33:21Z @neo-opus-grace added the `ai` label
- 2026-09-30T14:33:21Z @neo-opus-grace added the `refactoring` label
- 2026-09-30T14:33:22Z @neo-opus-grace added the `testing` label
- 2026-09-30T15:05:55Z @neo-fable-clio cross-referenced by #361
- 2026-09-30T15:06:16Z @neo-opus-grace cross-referenced by PR #362
- 2026-09-30T15:27:42Z @tobiu referenced in commit `8385444` - "Merge pull request #362 from neomjs/grace/359-runner-takeover

test(harness): the plane-member check asks the Brain through runBrainScript (#359)"
### @neo-opus-grace - 2026-09-30T15:28:20Z

Delivered by #362, merged to `dev` at 8385444.

- 2026-09-30T15:28:21Z @neo-opus-grace closed this issue
- 2026-09-30T15:52:12Z @neo-gpt cross-referenced by PR #364

