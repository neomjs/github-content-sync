---
id: 1961
title: A merged outside pull request gets the Engine's merge comment
state: CLOSED
labels:
  - enhancement
  - contributor-experience
  - ai
  - build
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-11-27T18:25:16Z'
updatedAt: '2026-11-27T19:50:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1961'
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
closedAt: '2026-11-27T19:50:23Z'
---
# A merged outside pull request gets the Engine's merge comment

Leaf B of neomjs/neo#22363: this repository's caller of the Engine's merge comment. Owned, so no curated labels: I file and build it, as the epic owner's cut assigned.

## Context

On every merged pull request from outside the team, neomjs/neo's `merge-comment.yml` posts one comment: the author's standing, which already counts merges across the Engine, Brain and the Institution, up to two curated issues to take next, and the badge. neomjs/neo#22363 records the operator's direction that all three repositories honor the badges program. neomjs/neo#22369 (merged as `c991ccb090`) made the workflow callable through `workflow_call`, the Engine half of that direction.

## The Problem

An outside contributor whose pull request merges here gets no merge comment. `.github/workflows/` has no caller, so the standing the Engine counts across all three repositories is never shown on a merge here.

## The Architectural Reality

Design authority: neomjs/neo#22363, "Leaves B (Brain) and C (Institution) are thin callers, each filed in its own repository".

- `claim-responder.yml` here is the precedent (#1241): a thin caller of the Engine's reusable workflow, pinned by SHA.
- The called workflow's header states the contract: the caller owns the `pull_request_target` trigger and the token (`contents: read`, `issues: read`, `pull-requests: write`; a called workflow cannot widen them), pins the file by commit SHA, and omits `concurrency:`. The job's filter (merged, not a bot, not a member) and its concurrency stay in neomjs/neo.
- In a called run `context.repo` names this repository, so the comment, its reads and the next issues are this repository's. The org-wide count and the badge reads name their own repository. neomjs/neo#22369's sibling-call spec arm pins this.

## The Fix

Add `.github/workflows/merge-comment.yml`. It triggers on `pull_request_target` with `types: [closed]`, grants those three permissions, and has one job: `uses: neomjs/neo/.github/workflows/merge-comment.yml@c991ccb0907504e676c477a6a60e6c5d020dea64`. It has no `if:` and no `concurrency:` of its own.

## Acceptance Criteria

- [ ] The caller has exactly that trigger, those permissions and that pin, and no `if:` or `concurrency:`.
- [ ] The pin resolves: `merge-comment.yml` at neomjs/neo `c991ccb090` accepts `workflow_call`.
- [ ] Before the merge, the operator adds this path to the repository's `pull_request_target` Actions policy (neomjs/neo#21069); enforcement begins on 2 December.
- [ ] Post-merge: the next merged outside pull request here gets the comment, or its run says why not.

## Out of Scope

- The comment's words, filter and modules, which live in neomjs/neo.
- A dispatch preview: the `pr` and `post` inputs belong to the called file's own `workflow_dispatch`, which a caller cannot pass.

## Related

neomjs/neo#22363 (leaf A) · neomjs/neo#20892 (epic) · neomjs/neo#21069 (Actions policies) · #1241 (the claim responder's caller)

Live latest-open + closed sweep: checked the latest open issues (155 open, red band; the leaf is the shape neomjs/neo#22363 prescribes for each repository) and an open + closed search (`merge comment`) at 2026-11-27T18:16Z; no equivalent. A2A: I hold leaves B and C by the epic owner's cut; no other claim.

Origin Session ID: 43879567-327d-43e7-bf0a-06359ab02052

Retrieval Hint: "merge comment thin caller workflow_call sibling repository standing badge"


## Timeline

- 2026-11-27T18:25:17Z @neo-opus-grace added the `enhancement` label
- 2026-11-27T18:25:18Z @neo-opus-grace added the `contributor-experience` label
- 2026-11-27T18:25:18Z @neo-opus-grace added the `ai` label
- 2026-11-27T18:25:18Z @neo-opus-grace added the `build` label
- 2026-11-27T18:25:18Z @neo-opus-grace added the `agent-os` label
- 2026-11-27T18:25:27Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-27T18:27:24Z @neo-opus-grace cross-referenced by PR #1962
- 2026-11-27T19:50:23Z @tobiu referenced in commit `d8b841c` - "feat(build): a merged outside pull request gets the Engine's merge comment (#1961) (#1962)

Leaf B of neomjs/neo#22363. A thin caller of neomjs/neo's merge-comment
workflow, pinned to neomjs/neo#22369's merge commit: it owns the
pull_request_target trigger and the token, as claim-responder.yml does. The
filter, the concurrency and the words stay in the Engine. A called run's
context.repo is this repository, so the comment and its next issues are
Brain's, while the standing still counts all three repositories."
- 2026-11-27T19:50:24Z @tobiu closed this issue
- 2026-11-27T19:53:54Z @neo-opus-grace cross-referenced by #22363

