---
id: 846
title: The onboardPeer two-process arm is red on dev; the base comparison hides it
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-grace
createdAt: '2026-10-04T16:15:57Z'
updatedAt: '2026-10-04T16:55:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/846'
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
closedAt: '2026-10-04T16:55:15Z'
---
# The onboardPeer two-process arm is red on dev; the base comparison hides it

## Context

Ada reported it, and Vega flagged it before her: on their hosts, `onboardPeer.spec.mjs`'s arm "two independent Node CLI processes observe one owner process and an idempotent second start" fails. `firstStart.instanceHome` is undefined at `:626`. Brain Unit still reports green.

Verified on CI:
- On #845's run (`37211546422`), `suite (head)` lists the arm as failed.
- On #839's own run (`37202140210`), both `suite (base)` and `suite (head)` fail it with the same `TypeError` at `:626`.

The job passes only because the base fails the arm too, so the head-vs-base comparison never shows it.

## The Problem

The arm stopped testing anything: every run fails before its first assertion. The failure is now two causes deep. One of them makes a unit arm read GitHub over the network.

## The Architectural Reality

The arm (`test/playwright/unit/ai/scripts/fleet/onboardPeer.spec.mjs:553`) runs the long-lived Fleet owner in-process (`startFleetBridgeServer`). It drives the owner from two Node CLI processes, which call `defineAgent` → `setRepo` → `startAgent`. The default `startAgentProvisioned` serves the start.

1. **The fixture repository is not a repository.** `:575` creates `repoPath/.git` as an empty directory. Once the start passes the identity gate, it fails at `git rev-parse --absolute-git-dir` against that directory. That probe runs in `convergeSeatGitIdentity` (`ai/services/fleet/seatGitIdentity.mjs:297`, since #839) and also lives in the seat-hook projector (`ai/scripts/lifecycle/hooks/projectSeatHooks.mjs:1144`). The base before #839 already failed the arm with the same symptom.
2. **Since #839, the start resolves the seat's Git identity first** (`startAgentProvisioned.mjs:282`). The arm declares none, so `resolveSeatGitIdentity` reads `https://api.github.com/user` with the fixture PAT `ghp_fixture_only`. The answer is 401, so the state is `unknown` and the start is refused. A declared identity resolves with no forge read (`seatGitIdentity.mjs:207`).

## The Fix

Test-only, in that arm:
- Make the fixture a real repository: `fs.mkdirSync(repoPath, {recursive: true})` plus `git init -q`, which stays local and hermetic.
- Declare the seat's identity in its define: `gitName`, `gitEmail`. The bridge carries both (`FleetControlBridge.defineAgent`).

Checked locally on dev `dbd35bc2` plus #845: the spec went from 36 passed and 1 failed to 37 passed.

## Acceptance Criteria

- [ ] AC-1: the arm passes locally and in CI's `suite (head)` with no network read: its define declares the identity, and its fixture repository is a real one.
- [ ] AC-2: red-first. The arm fails on dev with `instanceHome` undefined and passes with the fix. Reverting either half alone fails it again (a mutation run per half).
- [ ] AC-3: no production source changes.

## Out of Scope

- The head-vs-base comparison that hid this. Brain Unit is red on dev's own base across many specs; #201 and Ada's base-suite defect-note own that.
- How identity resolution behaves. #829 and #839 settled it.

## Related

#829 / #839 (the identity gate) · #201 (the retained unit suite in CI) · #845 (the run that showed it).

Sweeps: live latest-open sweep of the 20 newest open Brain issues at 2026-10-04T16:15Z, plus an owner-wide `onboardPeer` search: no equivalent. A2A lane-claims from the last hour: none on this scope. Memory Core rationale query: no prior decision. Own open assignments (#684, #18, #54): unrelated. Structure map: N/A, test-only.

Origin Session ID: 8a9af488-83ca-46fa-8ad4-41448d022b45

Retrieval Hint: "onboardPeer owner transport arm fixture .git identity gate api.github.com"

## Timeline

- 2026-10-04T16:15:58Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-04T16:15:59Z @neo-opus-grace added the `bug` label
- 2026-10-04T16:15:59Z @neo-opus-grace added the `ai` label
- 2026-10-04T16:15:59Z @neo-opus-grace added the `testing` label
- 2026-10-04T16:17:28Z @neo-opus-grace cross-referenced by PR #847
- 2026-10-04T16:22:30Z @neo-fable cross-referenced by #848
- 2026-10-04T16:55:15Z @tobiu referenced in commit `166f17f` - "test(fleet): the onboardPeer two-process arm starts on a real repository and a declared identity (#846) (#847)

The fixture's empty .git failed the start's git-dir probe, and since #839 the undeclared identity sent the start to api.github.com with a fixture PAT (401, refused). A git init repository and a declared gitName/gitEmail keep the arm hermetic."
- 2026-10-04T16:55:15Z @tobiu closed this issue

