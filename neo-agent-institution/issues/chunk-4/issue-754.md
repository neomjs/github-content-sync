---
id: 754
title: 'The pack stage fails on Node 24.22: cpSync refuses a shared directory'
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - build
  - regression
assignees:
  - neo-opus-ada
createdAt: '2026-10-19T17:48:57Z'
updatedAt: '2026-10-19T18:31:41Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/754'
author: neo-opus-ada
commentsCount: 0
parentIssue: 732
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-19T18:31:41Z'
---
# The pack stage fails on Node 24.22: cpSync refuses a shared directory

## Context

Institution CI's `Isolated Institution` job fails two `harness/pack.spec.mjs` arms on runners whose toolcache Node is 24.22.0, and passes on 24.21.0. `ci.yml` sets only `node-version: 24`, so whether a run goes red depends on the runner image, not the commit:

| Run | Commit | Node | Result |
|---|---|---|---|
| dev push, run 39289894420 | `f91b92dd` (#751's merge) | 24.22.0 | 2 failed |
| dev push, run 39293503164 | `32d3e340` (#750's merge) | 24.21.0 | green |
| #753, job 118201825867 | `42b279f` | 24.22.0 | 2 failed, the same two arms |

Maryam (CI hold on #753) and I found it independently.

## The Problem

Node 24.22.0 (2026-10-14) ships nodejs/node#64124, "fs: throw on existing dir in cpSync with errorOnExist". `copyTree` stages each owner's trees with `fs.cpSync(…, {errorOnExist: true, force: false, recursive: true})`, and both owners stage `src`: the Brain stages `ai` and `src`; the product stages `apps/agentos`, `dist/development/css` and `src`. The product's `src` copy now throws `ERR_FS_CP_EEXIST` on the existing `stage/src` directory itself. The wrapper reports it as `pack: two owners stage the same path under src`.

The same break stops `predist` (`node pack.mjs`, which runs on the build host's Node) on any host whose Node carries #64124. No FM candidate can be built there.

## The Architectural Reality

- Design authority: the comment above `copyTree` in `harness/pack.mjs`: "Two owners may share a directory in the stage (both carry a `src/`), never a file: an existing target is a collision between owners and fails the copy instead of letting the later one win."
- `pack.spec.mjs` pins both halves of that sentence in `stageOwners keeps provenance across the shared src tree: each owner is scanned from the files IT copied, and a merged scan is the red control`: the shared `src` merges, and a file both owners stage is refused.
- `copyTree` is the only `errorOnExist` user. `afterPack.cjs` (`cpSync`) and `runtimeGeneration.mjs` (`fs.cp`) pass none, so #64124 does not reach them.

## The Fix

- `copyTree` drops `errorOnExist`, keeps `force: false`, and refuses a collision itself. Its filter receives `(source, destination)`, and a shipped file whose destination exists throws `pack: two owners stage the same path: <stage-relative path>`, the message `copyFile` already uses. The `ERR_FS_CP_EEXIST` catch goes.
- `ci.yml`: both `actions/setup-node` steps add `check-latest: true`. Every job then runs the newest Node 24 patch, so a Node change reds every run at once instead of the runners whose image happens to carry it.

## Acceptance Criteria

- [ ] AC-1: On Node 24.22.0, `harness/pack.spec.mjs` passes, including `the cut-path control: themes built from Engine A, a stage that resolved Engine B fails, and no receipt is written; the same stage at A passes` and the shared-`src` arm above, whose second half still refuses a file both owners stage. The CI job log names the Node version.
- [ ] AC-2: Both `ci.yml` jobs resolve the newest Node 24 patch (`check-latest: true`), and their `Set up Node.js` lines show it.

## Out of Scope

- Other Node lines in CI. The fix stops depending on `errorOnExist`, so it holds on any line, but CI keeps testing Node 24 only.
- `afterPack.cjs` and `runtimeGeneration.mjs`: neither passes `errorOnExist`.

## Related

- Parent: #732. Found on #753 (Resolves #752), which merges dev again once this lands.
- nodejs/node#64124.

Sweeps: the latest 20 open issues at 2026-10-19T17:47:31Z, the last 30 A2A messages across read states, and a Memory Core query on the symptom. None found an equivalent ticket, a competing claim or a prior decision. Structure map: N/A (existing file, no placement).

Origin Session ID: 59bb04a0-8c50-4155-9b78-4b07ee58053a
Retrieval Hint: "Node 24.22 cpSync errorOnExist pack stage EEXIST shared src"

## Timeline

- 2026-10-19T17:48:57Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-19T17:48:59Z @neo-opus-ada added the `bug` label
- 2026-10-19T17:48:59Z @neo-opus-ada added the `agent-os` label
- 2026-10-19T17:49:00Z @neo-opus-ada added the `ai` label
- 2026-10-19T17:49:00Z @neo-opus-ada added the `build` label
- 2026-10-19T17:49:00Z @neo-opus-ada added the `regression` label
- 2026-10-19T17:49:03Z @neo-opus-ada added parent issue #732
- 2026-10-19T17:53:36Z @neo-opus-ada cross-referenced by PR #755
- 2026-10-19T18:31:37Z @tobiu referenced in commit `d4528ba` - "fix(harness): the pack stage checks a file collision itself, so the owners' shared src/ merges on Node 24.22 (#754) (#755)

Node 24.22 ships nodejs/node PR 64124: cpSync with errorOnExist now refuses an existing destination directory, so the product's src/ copy failed on the Brain's src/. copyTree drops errorOnExist and its EEXIST catch; its filter refuses a shipped file whose destination exists. CI's setup-node steps resolve the newest Node 24 patch (check-latest), so a Node change reaches every job at once."
- 2026-10-19T18:31:42Z @tobiu closed this issue

