---
id: 234
title: Request the declared reviewer when a PR's required checks turn green
state: CLOSED
labels:
  - enhancement
  - ai
  - build
  - model-experience
assignees:
  - neo-opus-ada
createdAt: '2026-10-29T00:10:57Z'
updatedAt: '2026-10-29T00:39:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/234'
author: neo-opus-ada
commentsCount: 2
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
closedAt: '2026-10-29T00:39:56Z'
---
# Request the declared reviewer when a PR's required checks turn green

> **Closed as not planned, 2026-10-29: superseded by neomjs/neo-agent-brain#759.** The design below took a GitHub review request to be an agent seat’s wake. It is not; #759’s holder wakes (#761, default-off) are. Evidence: https://github.com/neomjs/neo-agent-skills/issues/234#issuecomment-6484596553

## Context

`pull-request/references/ci-green-review-routing.md` §3 lets an author request the primary reviewer only once current-head CI is green. Nothing tells the author when that happens. The Claude desktop app's Auto-fix wakes a session on CI failures, merge conflicts and review comments, not on green, and Codex and Antigravity seats have no CI monitor at all. So a green PR waits for its author's next turn before anyone is asked to review it.

The operator put it on Brain PR #1343 (2026-10-29): "auto-fix only tells you when it fails, but not when CI turns green."

## The Problem

A Memory Core recall on the symptom finds the same stall across seats:

- 2026-10-05, Ada, Institution #567: a green status promised with no event to act on it.
- 2026-10-10, Ada, `neo-agent-skills#163`: the operator flagged that no reviewer was assigned; CI was already green.
- 2026-10-10, Clio, Institution #659: a missed wake from a misconfigured wait state.
- 2026-10-12, Grace: a monitor that fired on PR problems but not on green CI, replaced by a manual check.
- 2026-10-29, Ada, Brain #1343: the operator reported green.

Each costs an idle green PR or an operator nudge. GitHub already observes the transition; we leave it to agent memory.

## The Architectural Reality

- `ci-green-review-routing.md` §2–§3 owns author-side routing: exactly one `primary-reviewer` via `manage_pr_reviewers`, after green, plus one targeted A2A. Its §5 anti-pattern forbids requesting while CI is pending, because the request itself is the reviewer's wake.
- `.github/workflows/reusable-dependabot-automerge.yml` is the sibling precedent. It is a reusable workflow that acts on PR state and runs in the consumer with the consumer's own token.
- #154's avoided trap binds here: "Merging on 'all checks green' computed by the workflow itself — a second readiness predicate (#76's class)." GitHub's head-bound `mergeStateStatus` is the predicate instead. It reads `CLEAN` only when the required checks pass and the PR is mergeable (`neo`'s `review-admission/mergeability` status included, where it is required). Brain PR #1343 moved from `BLOCKED` to `CLEAN` at green.
- #14 (epic) makes this repository the source of truth for reusable PR-governance workflows with minimal pinned callers. This fits that shape.
- Design authority: n/a — nothing that exists is called wrong; this adds a trigger for a transition the routing rule already defines.

## The Fix

1. Add `.github/workflows/reusable-review-at-green.yml` (`workflow_call`). The caller runs it on `workflow_run: completed` of its CI workflows. For each same-repo, non-draft PR whose head equals the run's head, it reads `mergeStateStatus` and retries briefly while that is `UNKNOWN`. On `CLEAN`, with no review request and no review on the PR yet, it requests the login on the body's `Review at green: @login` line. In every other case it is a no-op, and it never checks out the PR.
2. Add the caller to `neo-agent-brain` first, then to the other consumers.
3. Update `ci-green-review-routing.md`:
   - At PR open, the author chooses the reviewer by the existing family → team → liveness → outbox order and writes the `Review at green:` line.
   - The targeted A2A goes out wake-suppressed, because GitHub's request at green is the wake.
   - The Pending branch's park-and-recheck steps shrink to the fallback for repositories without the caller.

   Read `create-skill`'s `skill-authoring-guide.md` before editing it.

Implementation notes, from the `reusable-dependabot-automerge.yml` precedent:

- The logic lives in `scripts/review-at-green.mjs`: a pure `decideReviewAtGreen` plus a `gh` action. It runs from the workflow's own commit through a sparse checkout of `job.workflow_repository` at `job.workflow_sha`, never from the PR head.
- `scripts/test-reusable-review-at-green.mjs` joins `npm test`. It runs fixtures through the real decision, the action against a recorded `gh`, a CLI arm through a stub `gh` on PATH, and workflow-contract negatives that must each turn red.
- The caller's `workflow_run.workflows` list must name every workflow that emits a required context. Otherwise the last green check has no trigger.
- Consumers pin a published release tag (`@vX.Y.Z`), so Brain adoption follows a release.

## Acceptance Criteria

- [ ] AC-1: On a PR with a `Review at green:` line, the declared login is requested once the PR's `mergeStateStatus` is `CLEAN` at the current head, and never while it is pending or failing.
- [ ] AC-2: The workflow requests nothing when a request or a review already exists, on fork or draft PRs, or when the declared login is the PR author. Reruns are idempotent.
- [ ] AC-3: This repository calls the workflow through a local caller, and a live PR here shows the request made by the workflow at green, with no author turn.
- [ ] AC-4: `ci-green-review-routing.md` documents the declare-at-open path, keeps the manual path only as the fallback, and does not grow in loaded bytes.

## Post-Merge Validation

- [ ] After the next release, Brain calls the workflow, and a live Brain PR shows a request made by it that reaches the declared seat without an author turn (its own Brain ticket).

## Out of Scope

- Re-review requests after a review-response push; the same mechanism could follow.
- Choosing the reviewer automatically. The author's family, team and liveness judgment stays with the author.
- The desktop app's Auto-fix event set, which is not Neo substrate.

## Avoided Traps

- **A plane-side "CI green" wake to the author:** it costs an author turn per PR plus plane machinery, and it leaves harnesses without a monitor uncovered.
- **Requesting at open:** the request is the reviewer's wake (§5), so a pending-CI request makes the reviewer hold.
- **Computing green in the workflow:** that would be a second readiness predicate (#154, #76). `mergeStateStatus` is GitHub's own, bound to the head.

## Related

#14, #154, #76. `neo`'s `.github/workflows/review-admission-mergeability.yml` (mergeability status). Trigger: Brain PR #1343.

Live latest-open + closed sweep: checked the latest 20 open issues here at 2026-10-29T00:09:39Z, plus closed searches for "reviewer green", "review request CI" and "workflow_run reviewer". Read the same-surface hits #14 (open) and #154 (closed; its Avoided Traps shaped the predicate above). No equivalent ticket or prior exclusion found.
A2A sweep: the 15 most recent messages (all read-states) hold no claim on this surface.
MC sweep: "reviewer request waits for CI green, author must remember to seat reviewer after checks pass, deferred review request activation", 8 results, the five incidents above and no prior decision.
Own-assignment sweep: 1 open (#102), not overlapping.
Meta-skill sweep: the change edits a payload, not a `SKILL.md` router, and shrinks it.

Ownership: self-assigned to @neo-opus-ada, to start after Brain PR #1343 lands. Adopting it into #14 is the epic owner's call.

Origin Session ID: 6cfc6a71-3d75-4616-87fc-b8ba06833824
Retrieval Hint: "green PR waits for author turn before reviewer request; review at green workflow"



## Timeline

- 2026-10-29T00:10:58Z @neo-opus-ada added the `enhancement` label
- 2026-10-29T00:10:59Z @neo-opus-ada added the `ai` label
- 2026-10-29T00:10:59Z @neo-opus-ada added the `build` label
- 2026-10-29T00:10:59Z @neo-opus-ada added the `model-experience` label
- 2026-10-29T00:11:07Z @github-actions added the `needs-design-authority` label
- 2026-10-29T00:11:36Z @neo-opus-ada assigned to @neo-opus-ada
### @neo-opus-ada - 2026-10-29T00:20:38Z

Handover (session sunset, scope solo-refresh): intake is done and implementation has not started.

- The plan and the precedent's conventions are in the body: **Implementation notes**, the AC-3 split, and **Post-Merge Validation**.
- Pickup: branch `ada/234-review-at-green` from `origin/dev` (`6f1bb5b`) in the Skills clone. Start with `scripts/review-at-green.mjs` and its test, mirroring `scripts/dependabot-automerge-handoff.mjs` and `scripts/test-reusable-dependabot-automerge.mjs`.
- Before editing `ci-green-review-routing.md`: `create-skill`'s `skill-authoring-guide.md` and `turn-memory-pre-flight`.
- Owner: @neo-opus-ada, carried by the next session.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-10-29T00:26:58Z @github-actions removed the `needs-design-authority` label
- 2026-10-29T00:39:18Z @neo-opus-ada cross-referenced by #759
### @neo-opus-ada - 2026-10-29T00:39:43Z

Closed as not planned: superseded by neomjs/neo-agent-brain#759. My filing never read the plane side, and three of its premises fail:

1. **The wake.** `pull-request-workflow.md` §6.2 pairs the GitHub request (the seat) with a targeted A2A (the wake). Agent seats do not read GitHub notifications: the heartbeat's notification ingestion has been off since the operator's 2026-09-24 ruling. Fix step 3 would have suppressed the only wake, so the reviewer would never have woken.
2. **The plane-side wake exists.** #759, re-planned on the operator's direction on 10-13, owns this outcome. #761 shipped its holder wakes, default-off: `route-review` wakes the author of a green PR with no review requested, and `review` wakes a requested reviewer once CI is green. The first Avoided Trap above is that design. Emmy's 10-28 analysis on #759 asks that the installed enablement and delivery state be established "before adding another mechanism".
3. **AC-3 could not be met before merge.** `workflow_run` triggers only from a workflow file on the default branch, so the PR could not have shown its own caller working.

The workflow loses in both states. While holder wakes are off, a GitHub request wakes no agent seat. Once they deliver, a request made when the PR opens gets the reviewer woken at green, with no author turn and no workflow, because `holderOf` gates `review` on green. That input is on #759: https://github.com/neomjs/neo-agent-brain/issues/759#issuecomment-6484589708

The branch `ada/234-review-at-green` had no commits and is deleted.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


- 2026-10-29T00:39:56Z @neo-opus-ada closed this issue

