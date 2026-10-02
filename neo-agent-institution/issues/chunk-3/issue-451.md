---
id: 451
title: Pin the merged Brain setup and Fleet contracts
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - build
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-02T16:27:34Z'
updatedAt: '2026-10-02T16:49:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/451'
author: neo-gpt-emmy
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
blocking:
  - '[ ] 440 The setup card''s run and re-check actions reach the vessel''s effect channel, and the first completed run records its density'
closedAt: '2026-10-02T16:49:10Z'
---
# Pin the merged Brain setup and Fleet contracts

## Context

The operator merged the reviewed Fleet changes on 2 October. Institution dev `87e4f1c4a75b1c292e3aa3e093b0a4430b44e12c` still pins Brain `a9dd22ff3dd7c01ba8beb721402f6ec1a4fc874f`. The merged Brain now reaches `447d96e66470d9b3865f95ee243b58b3215e0254`, making the shared setup-effect orchestration available to #440 and the seat-aware GitLab repository defaults available to #448.

## The Problem

The product's installed dependency cannot expose these merged capabilities until its manifest, lock and explicit Brain CI checkout advance together. Reusing only an external checkout would test a different product from the one the lock installs.

## The Architectural Reality

The frozen Brain range contains five first-parent changes: neomjs/neo-agent-brain#752, neomjs/neo-agent-brain#753, neomjs/neo-agent-brain#756, neomjs/neo-agent-brain#758 and neomjs/neo-agent-brain#765. There are no changed shared `src/**` files or Brain package/lock dependency declarations in that range.

The consumed start/restart result changes to a named `{status:'rejected', reason}` outcome. Its paired consumer is already merged in #444: `FleetLifecycleIntentAdapter.handleFleetLifecycleIntent` handles that outcome before declaring success. The GitLab change derives missing clone URLs at the seat-aware manager boundary; explicit repository entries remain valid. The new `ai/services/fleet/setupOrchestration.mjs` is the #440 consumer's dependency, not an effect invocation by this pin.

## The Fix

Advance only the Brain reference in `package.json`, `package-lock.json`, and `.github/workflows/ci.yml` to `447d96e66470d9b3865f95ee243b58b3215e0254`. Keep Engine `93769448934166a8c98b4d99eccda4c3d347caeb` unchanged. Reinstall from the lock and verify the existing Institution suites and the paired rejection behavior against the selected Brain.

Prescription checked: the package manifest/lock and explicit Brain CI checkout own this selection; no runtime configuration or product-layer copy is needed.

## Contract Ledger

| Target surface | Source of authority | Behavior | Edge / fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Brain dependency and CI checkout | merged Brain commit above | all three references select the same immutable commit | no floating dev or unmerged source | PR pin census | exact references + lock installation |
| start/restart rejected outcome | Brain FleetControlBridge and merged Institution #444 | existing consumer retains rejected state/reason | success and thrown-failure paths retain existing semantics | existing JSDoc unchanged | existing lifecycle adapter controls |
| Engine pin | current Institution manifest | remains 9376944 | no pending Engine patch is absorbed | PR scope | diff and resolved graph |

## Acceptance Criteria

- [ ] Manifest, lock and CI select the exact frozen Brain SHA, and Engine remains unchanged.
- [ ] The locked install resolves that Brain revision and exposes setupOrchestration.
- [ ] Existing isolated and explicit-Brain contract checks pass; the refusal consumer's existing controls remain green.
- [ ] The PR distinguishes source validation from package build and installed acceptance.

## Out of Scope

#440's effect-channel implementation; #448's add form; the later open-work producer; an Engine bump; native hook projection; package replacement or seat interruption.

Decision Record impact: none — dependency selection only.

Related: #440 · #448 · #444 · #442.

Sweeps: latest 20 open Institution issues plus all-state recent A2A checked at 16:27 UTC; no competing pin issue or claim. Own assignment is #42 (measurement only); previous pin #442 is closed after merge. Memory Core query on the missing setup-effect dependency returned unrelated initialization records, not a prior decision. Brain structure map has run in this session; no module is added or moved.

Origin Session ID: 3acb1755-5285-4f3a-a74a-dae637bb629d
Retrieval Hint: `a9dd22f..447d96e setupOrchestration Fleet refusal GitLab seat repositories`


## Timeline

- 2026-10-02T16:27:34Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-02T16:27:36Z @neo-gpt-emmy added the `enhancement` label
- 2026-10-02T16:27:36Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-02T16:27:36Z @neo-gpt-emmy added the `ai` label
- 2026-10-02T16:27:37Z @neo-gpt-emmy added the `build` label
- 2026-10-02T16:27:54Z @neo-gpt-emmy marked this issue as blocking #440
- 2026-10-02T16:34:06Z @neo-gpt-emmy cross-referenced by PR #452
- 2026-10-02T16:40:08Z @neo-gpt-emmy cross-referenced by #453
- 2026-10-02T16:49:10Z @tobiu referenced in commit `99e2acf` - "feat(deps): carry merged setup and Fleet contracts (#451) (#452)"
- 2026-10-02T16:49:11Z @tobiu closed this issue
- 2026-10-02T18:03:37Z @neo-opus-ada cross-referenced by #460

