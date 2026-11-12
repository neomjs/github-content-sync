---
id: 366
title: 'PR-loop handoffs ride the Fleet''s open-work wakes, not seat DMs'
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
  - needs-design-authority
assignees:
  - neo-opus-ada
createdAt: '2026-11-12T01:29:20Z'
updatedAt: '2026-11-12T04:33:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/366'
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
closedAt: '2026-11-12T04:33:01Z'
---
# PR-loop handoffs ride the Fleet's open-work wakes, not seat DMs

## Context

Operator, 2026-11-12 ~01:25Z, after I offered Claude Code's Auto-fix on Brain PR #1623 to bring me back when CI finished: "'turn on auto-fix' is absurdly pointless. you will idle out forever if CI turns green."

Operator, 2026-11-12 ~03:15Z (friction → gold): "so far our skills recommend to write peers a2a messages, in case they e.g. ask for a review. now that FM automatically does this, we should simplify our skills. less cognitive load for peers who no longer need to write, less noise for receivers, who don't get duplicates."

Specimen, Brain PR #1626: Maryam's seat was woken for one review by the Fleet's `[open-work:review]` wake on each head and by two DMs of mine (the review request and a head-moved note). Each DM said what a wake already said.

## The Problem

- **The CI wait names no trigger that fires on green.** `ci-green-review-routing.md` §3 Pending says to "park a concrete recheck trigger (watchdog wake, next turn, or after the next short lane)". A Claude seat follows the harness instead and arms Auto-fix, which wakes on red and on comments, never on green. Grace (10-12), Vega (10-05) and Ada (Brain #1343, #1623) stalled this way. #234 filed it and closed as superseded by `neomjs/neo-agent-brain#759`, but its routing-text step was never done.
- **The skills still ask for handoff DMs the Fleet now sends.**
  - `ci-green-review-routing.md` §3 Green step 3: a targeted DM to the reviewer.
  - `pull-request-workflow.md` §6.2: "send one matching wake".
  - §6.3, `review-response-protocol.md` §14 and `pr-review-guide.md` §10: commentId handoff DMs between author and reviewer.
  - Every one duplicates a wake, costing the writer a message and the reader a second wake.

## The Architectural Reality

The Fleet's open-work producer derives each open PR's next action from GitHub state (`ai/services/fleet/openWorkHolder.mjs`, `holderOf`) and wakes the holder once per head (`neomjs/neo-agent-brain#759`, PR #1610). Since #1613 it sends as `@system`.

| GitHub state of the head | Holder | Wake |
| :--- | :--- | :--- |
| CI red | author | `repair-ci` (names the failed checks) |
| a change request on the head | author | `address-changes` |
| approved, CI green, conflicting | author | `rebase` |
| CI green, reviewer(s) requested | each requested reviewer | `review` |
| CI green, no opinion on the head | author | `route-review` |
| approved, CI green, mergeable | operator | `merge` (FM surface, no seat wake) |
| merged | author | `post-merge` |

- **Coverage:** every GitHub repository the registry's seats work on (`wireFleetOpenWorkSource.mjs`, `githubSlugsOf`).
- **Not covered:**
  - GitLab merge requests (forge filter).
  - Outside-contributor PRs: the `rotation` role names nobody until `neomjs/neo-agent-brain#807`.
  - Lane claims, escalations, tie-breakers, declines, reroutes, operator-only asks, and Discussion reads.
- **The review-load coupling.** `who_is_online`'s review-load axis (`ai/services/memory-core/helpers/reviewLoadProjection.mjs`) counts reviewers' `review-posted` pings to any recipient. Those open and close a loop. It also counts authors' re-review pings addressed to the reviewer, which mark the loop `returned`.
  - The reviewer's quiet `AGENT:*` lifecycle broadcast stays, so the loops keep opening and closing.
  - Only the `returned` marker loses its author DM, and the `review` wake now carries that fact.
- **No critical gate changes.** Gate 6 ("notify peers via add_message") is met by the quiet broadcast.
- **Payload rules.** Every file touched is a skill-loaded payload. A turn-memory pre-flight runs at implementation, and `create-skill`'s `skill-authoring-guide.md` is read before editing. The `SKILL.md` routers are untouched.

## The Fix

1. **`ci-green-review-routing.md` §3.**
   - Green: `manage_pr_reviewers` only. The GitHub request is the producer's input, and its `review` wake reaches the reviewer, so no targeted DM.
   - Pending: the CI wait is the Fleet's `route-review` / `repair-ci` wake for the head, never a harness PR monitor (Auto-fix wakes on red and comments, never green) or a self-scheduled poll.
   - §5: one row for each anti-pattern (monitor as CI wait; DM duplicating a wake).
2. **`pull-request-workflow.md`.**
   - §6: one clause, the CI wait is the Fleet's wake.
   - §6.2: drop "send one matching wake".
   - §6.3: the author's handback is GitHub's re-request plus the review-response comment.
3. **`review-response-protocol.md` §14 and `pr-review-guide.md` §10:** retire the commentId handoff DMs. The PR thread and the quiet `AGENT:*` lifecycle broadcast stay. `post-review-pickup` references follow.
4. **Keep list, stated once:** where a seat still writes a targeted A2A message (the "Not covered" list above).
5. A minor version bump, since a rule changes.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `pull-request/references/ci-green-review-routing.md` §1, §3 Green / Pending, §4, §5 | operator 2026-11-12 (both quotes above); `openWorkHolder.mjs` | request and re-request on GitHub only; the CI wait is the Fleet’s `route-review` / `repair-ci` wake; no handoff DM | a hand-off the Fleet does not carry still uses A2A | the payload | `npm run lint`, `npm test`; post-merge AC |
| `pull-request/references/pull-request-workflow.md` §6, §6.2, §6.3 | same | the CI wait points to §3; no "matching wake"; the keep list, stated once | none | the payload | same |
| `pull-request/references/review-response-protocol.md` §14, `pr-review/references/pr-review-guide.md` §10, the round-2 and follow-up templates | same | the formal review and GitHub’s re-request are the hand-off; the woken seat reads with `since_comment_id` / `last_n` | tie-breaker, escalation and decline still A2A the `commentId` | the payloads | same |
| `post-review-pickup` description and its `skills.manifest.json` mirror | same | trigger "author handback" replaces "author commentId handoff" | none | router + manifest | `npm run lint` (manifest coherence) |

## Acceptance Criteria

- [ ] No skill payload asks a seat to DM a handoff the producer sends (review request, head moved, re-review, changes addressed, CI result, merge-ready). Each names the GitHub action that feeds the producer instead.
- [ ] §3 Pending names `route-review` / `repair-ci` as the CI wait and rules out harness monitors and self-scheduled polls. §5 carries both rows.
- [ ] The keep list is stated once.
- [ ] `npm run lint` (combined budgets included) and `npm test` pass, and loaded bytes go down.
- [ ] *(Post-merge, on an FM build carrying `neomjs/neo-agent-brain#1610` and #1613)* A Claude-authored PR runs its review loop with no handoff DM: the `review` wake reaches the reviewer, and `address-changes` / `route-review` reach the author.

## Out of Scope

- **Gate 6's lifecycle broadcasts** (`pr-opened`, `review-posted`, merged). They still feed the review-load axis. Retiring them needs that axis re-derived from the open-work snapshot first, and a critical-gate change goes through a Discussion.
- The wake itself (`neomjs/neo-agent-brain#759`, #1609, PR #1610, PR #1613), the desktop app's prompt, and reviewer-side CI holds (`pr-review/audits/ci-security-audit.md`).

## Avoided Traps

- **A Claude-only rule in turn-loaded substrate.** The trigger is harness-neutral. Auto-fix is the example a harness pushes, named in the payload that governs the wait.
- **A GitHub-side trigger again (#234's workflow).** Seats do not read GitHub notifications; the Fleet's wake is the owned path.
- **Deleting the broadcasts in the same PR.** That would blind the review-load axis.

## Related

- `neomjs/neo-agent-brain#759` (epic), #1609 / PR #1610 (wakes on), #1536 (`repair-ci` names checks), PR #1613 (`@system` sender), #807 (rotation recipient)
- #234 (closed: the same CI-wait gap)

Live latest-open + closed sweep: latest 20 open (35 open, orange band; `#364`..`#51`) plus a closed search (`CI wait recheck trigger wake review request green`; #234 read in full) at 2026-11-12T01:30Z. Re-swept for the widening at 03:20Z: org-wide open searches `review DM duplicate wake` and `A2A handoff open-work`, plus a title scan of the open Skills queue; no owner.
A2A in-flight claim sweep: latest 12 messages across all read states at 01:30Z; no claim on this surface.
MC sweep: `author idles after CI turns green; nothing wakes the seat to request a reviewer; auto-fix monitor pointless`, 6 results; no later decision.
Own-assignment sweep: 4 open (#364, #282, #102, #298), none on this surface.
Structure map: N/A (no `.mjs` placed; skill payload text only).

Ownership: self-assigned, @neo-opus-ada.

Origin Session ID: a84af7d7-9095-4958-8288-503b774a5f6b (widened in c0922281-1f36-48c2-bd05-4180f854c2ca)
Retrieval Hint: "PR-loop handoffs ride the Fleet's open-work wakes: no handoff DMs, no harness monitor as the CI wait"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-12T01:29:20Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-12T01:29:22Z @neo-opus-ada added the `enhancement` label
- 2026-11-12T01:29:22Z @neo-opus-ada added the `ai` label
- 2026-11-12T01:29:22Z @neo-opus-ada added the `model-experience` label
- 2026-11-12T01:29:23Z @neo-opus-ada added the `agent-os` label
- 2026-11-12T01:40:42Z @neo-opus-ada cross-referenced by #367
- 2026-11-12T03:21:15Z @neo-opus-ada changed title from **The author's CI wait names no trigger that fires on green** to **PR-loop handoffs ride the Fleet's open-work wakes, not seat DMs**
- 2026-11-12T03:21:25Z @github-actions added the `needs-design-authority` label
- 2026-11-12T03:41:13Z @neo-opus-ada cross-referenced by PR #369
- 2026-11-12T04:17:14Z @neo-opus-ada referenced in commit `04add50` - "fix(pull-request): the no-DM rule covers only what the Fleet routes (#366)

Sophie's review (RA-1, RA-2): holderOf keeps the author the holder while a change request sits on the head, and review/route-review need green CI. So a handback with no new commit and a repository without CI keep their A2A, named once in pull-request-workflow 6.2's keep list. post-review-pickup 4 and 11 fold into the handback route."
- 2026-11-12T04:33:00Z @tobiu referenced in commit `146a930` - "feat(pull-request): the Fleet's wakes carry PR handoffs, not DMs (#366) (#369)

* feat(pull-request): the Fleet's wakes carry PR handoffs, not DMs (#366)

The skills stop asking seats to DM what the Fleet's open-work producer already wakes the holder with: the review request, the re-review handback, the reviewer's commentId hand-off. The CI wait is the Fleet's route-review / repair-ci wake, never a harness PR monitor. Gate 6's quiet lifecycle broadcasts stay; they feed who_is_online's review-load axis. Minor bump to 1.6.0.

* fix(pull-request): the no-DM rule covers only what the Fleet routes (#366)

Sophie's review (RA-1, RA-2): holderOf keeps the author the holder while a change request sits on the head, and review/route-review need green CI. So a handback with no new commit and a repository without CI keep their A2A, named once in pull-request-workflow 6.2's keep list. post-review-pickup 4 and 11 fold into the handback route."
- 2026-11-12T04:33:02Z @tobiu closed this issue

