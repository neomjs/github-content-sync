---
id: 471
title: Packaged smoke rejects the merged setup shell capabilities
state: OPEN
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-02T21:13:06Z'
updatedAt: '2026-10-02T21:13:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/471'
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
blocking: []
---
# Packaged smoke rejects the merged setup shell capabilities

## Context

The operator-requested runtime refresh reached packaged Electron smoke at `dev@999fb37`. The executable exited 1 although product first paint, both windows, Fleet round trips, forged-sender refusal, secret census and unforced teardown passed. The failing conjunct was `brain.fleetFromWindow.surfaceExact`.

## The Problem

`harness/main.mjs:1541` still expects six shell keys. Merged commit `e6bced4` (PR `#441`) added the six setup capabilities in `harness/preload.cjs:20-25`: `setupAnswer`, `setupCredential`, `setupEffect`, `setupEvaluate`, `setupPresets`, `setupProbe`. All four smoke probes observed the same twelve-key surface; the old expected list rejects it. This blocks package acceptance for the runtime refresh in #12.

## The Architectural Reality

The preload exposes named IPC affordances; the smoke independently checks exact equality across primary, popup, surviving primary and forged-origin windows. The forged sender must still be refused by the main-process handler. This is a stale diagnostic expectation, not a request to add or broaden capabilities.

## The Fix

Update only the existing expected shell-key list in `harness/main.mjs` to include the six merged setup names in sorted order. Preserve exact equality and every other smoke conjunct. No new file or abstraction is needed.

## Acceptance Criteria

- [ ] The packaged smoke accepts exactly the current twelve named shell keys in all four probes; removing a setup key or adding an unexpected key still fails exact equality.
- [ ] The rebuilt packaged executable exits 0 in the isolated default product profile, retaining forged-sender refusal, secret-free IPC, both-window behavior and clean process/port teardown.

## Out of Scope

Runtime IPC behavior, setup effects, Brain pins, installed seat/profile mutation and the pending `#464` effect-channel implementation.

## Avoided Traps

Do not derive the expectation from the observed object or weaken equality to subset membership: either would hide accidental API expansion. Do not install a package while treating the original exit-1 receipt as a pass.

Decision Record impact: none; restores verification of the already-merged named capability surface. Structure map: N/A, existing Institution shell diagnostic only; no Brain placement or new module.

## Freshness and Related

Related: #12

Live latest-20-open and open-PR sweep at 2026-10-02 21:13Z found no equivalent repair; the only open PR `#464` touches setup handlers but not this expected-key list. Latest 30 A2A rows contain no overlapping claim. Own-assignment sweep: only `#42`, a view-layer investigation, not this shell check. KB and two broad MC searches were inconclusive; an own-session exact-symbol sweep returned prior pin/debt work, no prior decision about this failure. Live source and the failing executable are the evidence.

Origin Session ID: 8d1cf4b5-75d2-4880-8358-873e0ac47fe0
Retrieval Hint: `999fb37` packaged smoke `surfaceExact` / `expectedShellKeys`.

## Timeline

- 2026-10-02T21:13:06Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-02T21:13:07Z @neo-gpt-emmy added the `bug` label
- 2026-10-02T21:13:08Z @neo-gpt-emmy added the `ai` label
- 2026-10-02T21:13:08Z @neo-gpt-emmy added the `testing` label
- 2026-10-02T21:20:06Z @neo-gpt-emmy cross-referenced by PR #472

