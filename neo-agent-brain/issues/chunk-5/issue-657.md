---
id: 657
title: 'A /proc path spins a Linux test worker forever, so Brain unit never exits'
state: OPEN
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-grace
createdAt: '2026-09-30T23:10:18Z'
updatedAt: '2026-09-30T23:10:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/657'
author: neo-opus-grace
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
# A /proc path spins a Linux test worker forever, so Brain unit never exits

## Context

The full Brain unit run never exits on CI. On #651's workflow (runs 36772724151 and 36777160574), the last test finishes at about 400 s. The run then waits until `--global-timeout` (600 s) ends it, and `chroma-teardown` never starts. That bound is a workaround: it costs each leg about 3.5 minutes and plants a top-level timeout error on both sides. The sighting was recorded as a defect-note, fingerprint `3478ccebc7b5d1e6`.

## The Problem

Both runs' JSON reports show the same thing:
- One worker's last test is `seatProvenance.spec.mjs › an unwritable trace never disturbs the boot`, started at 195.5 s and 200.0 s. It is recorded as `skipped` with a 0 ms duration and no skip annotation: interrupted at the global timeout, never finished.
- That worker's parallel slot runs nothing after it.
- The other three slots drain the queue by about 400 s. The runner then waits on the stuck worker, so the teardown project never starts.

**Root cause.** The arm calls `recordTrace('/proc/definitely-not-writable-by-this-test', …)`, which runs `fs.mkdirSync(<that>/.agents, {recursive: true})`.
- On Linux, Node's native recursive mkdir never returns for a missing child of `/proc`.
- Reproduced in `node:24-alpine`, CI's Node major: the call was killed after 8 s with no output, while the same call under `/tmp` returns in 1 ms.
- The loop is synchronous, so the 30 s test timeout can never fire.
- On macOS `/proc` does not exist and the call fails at once, which is why local full runs exit.

## The Architectural Reality

- The arm is `test/playwright/unit/ai/scripts/lifecycle/hooks/seatProvenance.spec.mjs:200` (from #328).
- `recordTrace` is `ai/scripts/lifecycle/hooks/seatProjectionCheck.mjs:246`. It first asks `tracked()` (`git ls-files` with that path as `cwd`, which fails fast), then runs the recursive mkdir.
- Its contract (never throws, never blocks) holds for every path a hook is handed: a checkout, never procfs.

## The Fix

The arm hands `recordTrace` a path below a regular file. `mkdirSync` then throws `ENOTDIR` at once, on every platform and as root (measured: 1 ms in the same container). Production code is unchanged.

## Acceptance Criteria

- [ ] AC-1: The arm's unwritable root is a path below a regular file in the test's scratch space. It still asserts that `recordTrace` reports `false`.
- [ ] AC-2 *(post-merge)*: A full Brain unit run on CI exits on its own. Neither report carries a global-timeout error, and `chroma-teardown` has a result.

## Out of Scope

- The run bound in #651. It stays as a safety net against the next hang, and its cut-short refusal keeps a hang from buying coverage.
- Guarding `recordTrace` against procfs: no hook target is one.

## Related

#650 / #651 (the full run that surfaced it), #328 (the arm's origin), #201.

Decision Record impact: none.

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-09-30T23:15Z; no equivalent found. Keyword sweep (`hang teardown`, `never exits`, `global-timeout`, `seatProvenance`): none.
MC sweep: "Brain unit run never exits on CI, chroma teardown never starts, worker hangs", 5 results, no prior decision found.
Own-assignment sweep: #650 is mine and adjacent. This is a different defect, found through #650's run.

Origin Session ID: 8c224931-7b3d-4cb5-a43d-86f1735f3636
Retrieval Hint: "recursive mkdir /proc hangs a Linux worker, unit run never exits"


## Timeline

- 2026-09-30T23:10:19Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-30T23:10:20Z @neo-opus-grace added the `bug` label
- 2026-09-30T23:10:20Z @neo-opus-grace added the `ai` label
- 2026-09-30T23:10:20Z @neo-opus-grace added the `testing` label
- 2026-09-30T23:13:20Z @neo-opus-grace cross-referenced by PR #658

