---
id: 366
title: The author's CI wait names no trigger that fires on green
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-12T01:29:20Z'
updatedAt: '2026-11-12T01:29:20Z'
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
---
# The author's CI wait names no trigger that fires on green

## Context

Operator, 2026-11-12 ~01:25Z, after I offered Claude Code's Auto-fix on Brain PR #1623 to bring me back when CI finished: "'turn on auto-fix' is absurdly pointless. you will idle out forever if CI turns green." And: "we added new FM messages for CI results. they will become wake messages inside the next FM instance update."

- **Auto-fix never fires on green.** Claude Code's Auto-fix wakes a session on CI failures, merge conflicts and review comments (the desktop app's own monitor description). After `gh pr create`, the desktop app tells the seat to offer it.
- **Green leaves one step undone.** On green the author has one step left: request one primary reviewer (`pull-request/references/ci-green-review-routing.md` §3 Green). Nothing the harness offers fires on that transition.
- **The Fleet already sends the signal.** Its open-work producer (`neomjs/neo-agent-brain#759`) sends the author `route-review` when the head's CI is green, and `repair-ci`, naming the failed checks, when it is red (`neomjs/neo-agent-brain#1536`). PR `neomjs/neo-agent-brain#1610` (`75c0b31a`, 2026-11-11 20:52Z) makes those messages wake their seat. It reaches seats with the next FM build.

## The Problem

`ci-green-review-routing.md` §3 Pending says to "park a concrete recheck trigger (watchdog wake, next turn, or after the next short lane)". It names no trigger that fires on green, and nothing in it warns off a harness monitor, so a Claude seat follows the harness's prompt instead.

The record shows the stall repeating. Grace (10-12), Vega (10-05) and Ada (Brain #1343 on 10-28, Brain #1623 on 11-12) each armed Auto-fix expecting it to bring them back on green. #234 filed the same gap on 10-29 and closed as superseded by `neomjs/neo-agent-brain#759`, the plane-side wake. Its Fix step 3, the routing text, was never done.

## The Architectural Reality

Design authority: `"Return to your PR at the recheck trigger and re-run gh pr checks <PR_NUMBER> before assigning a reviewer."` (`ci-green-review-routing.md` §3 Pending). The payload requires a trigger that brings the author back on green. `neomjs/neo-agent-brain#759`'s terminal predicate makes the Fleet's open-work wake that trigger: "Every open PR … wakes the seat that holds its next action once per change of holder".

- `pull-request-workflow.md` §6 (Definition of Done) is what a seat reads when it opens a PR, which is when the harness's monitor prompt fires.
- Both files are skill-loaded payloads. The change rewrites one step and adds one anti-pattern row and one clause, so loaded bytes stay roughly flat (turn-memory pre-flight at implementation).
- Meta-skill sweep: no `SKILL.md` router changes. `create-skill`'s `skill-authoring-guide.md` is read before editing.

## The Fix

1. In `ci-green-review-routing.md` §3 Pending, step 3: replace the trigger list with the Fleet's open-work wake for the head, `route-review` on green and `repair-ci` on red. State that a harness PR monitor is not a trigger (Claude Code's Auto-fix wakes on red and on comments, never on green), and neither is a self-scheduled poll. Add a §5 row: "Arming a harness PR monitor as the CI wait | It never fires on green, so the review request never goes out."
2. In `pull-request-workflow.md` §6, maintainers paragraph: one clause saying to wait for CI on the Fleet's open-work wake (`ci-green-review-routing.md` §3), never a harness monitor.
3. A minor version bump, since a rule changes.

## Acceptance Criteria

- [ ] §3 Pending names the Fleet's `route-review` / `repair-ci` wake as the recheck trigger and rules out a harness PR monitor and self-scheduled polls; §5 carries the anti-pattern row.
- [ ] `pull-request-workflow.md` §6 carries the clause. `npm run lint` (combined budgets included) and `npm test` pass, and loaded bytes grow by no more than the clause.
- [ ] *(Post-merge, after an FM build that carries `neomjs/neo-agent-brain#1610`)* A Claude seat that opens a PR does not arm Auto-fix, and the `route-review` wake brings it back on green to request its reviewer.

## Out of Scope

- The wake itself: `neomjs/neo-agent-brain#759`, `neomjs/neo-agent-brain#1609` and PR #1610.
- The desktop app's own prompt.
- Reviewer-side CI holds (`pr-review/audits/ci-security-audit.md`).

## Avoided Traps

- **A Claude-only rule in turn-loaded substrate.** The trigger is harness-neutral. Auto-fix is named as the example a harness pushes, in the payload that governs the wait.
- **A GitHub-side trigger again (#234's workflow).** Seats do not read GitHub notifications. The Fleet's wake is the owned path (#234's closing evidence).

## Related

- `neomjs/neo-agent-brain#759` (epic: own-work transitions reach the holder), `neomjs/neo-agent-brain#1609` / PR #1610 (open-work wakes on), `neomjs/neo-agent-brain#1536` (`repair-ci` names the failed checks)
- #234 (closed: the same gap; its GitHub-side design was superseded)

Live latest-open + closed sweep: latest 20 open (35 open, orange band; `#364`..`#51`) plus a closed search (`CI wait recheck trigger wake review request green`; #234 read in full, including its closing comments) at 2026-11-12T01:30Z. No open equivalent; #234's step 3 is the unclaimed remainder.
A2A in-flight claim sweep: latest 12 messages across all read states at 01:30Z; no claim on this surface.
MC sweep: `author idles after CI turns green; nothing wakes the seat to request a reviewer; auto-fix monitor pointless`, 6 results (the stalls cited above); no later decision.
Own-assignment sweep: 4 open (#364, #282, #102, #298), none on this surface.
Structure map: N/A (no `.mjs` placed; skill payload text only).

Ownership: self-assigned, @neo-opus-ada.

Origin Session ID: a84af7d7-9095-4958-8288-503b774a5f6b
Retrieval Hint: "author CI wait names no trigger that fires on green: Fleet open-work route-review wake, never Claude Code Auto-fix"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-12T01:29:20Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-12T01:29:22Z @neo-opus-ada added the `enhancement` label
- 2026-11-12T01:29:22Z @neo-opus-ada added the `ai` label
- 2026-11-12T01:29:22Z @neo-opus-ada added the `model-experience` label
- 2026-11-12T01:29:23Z @neo-opus-ada added the `agent-os` label
- 2026-11-12T01:40:42Z @neo-opus-ada cross-referenced by #367

