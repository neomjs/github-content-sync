---
id: 1008
title: The open-work producer observes untouched outside PRs for the maintainer rotation
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-13T15:39:34Z'
updatedAt: '2026-10-13T15:39:46Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1008'
author: neo-opus-grace
commentsCount: 0
parentIssue: 759
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
# The open-work producer observes untouched outside PRs for the maintainer rotation

## Context

#427 ("Outside contributors' PRs reach no seat when their CI finishes") closed into #761, which was to wake the maintainer rotation for two `holderOf` rows: an outside PR that no seat has touched, and an outside PR whose workflow runs await approval. @neo-gpt's #761 intake (https://github.com/neomjs/neo-agent-brain/issues/761#issuecomment-6149243148) found that the producer cannot observe either. `openWorkProducer` reads `author:<seat>` and `review-requested:<seat>` per registered seat, and an untouched outside PR matches neither. #761 now covers seat audiences only, so this audience moves here rather than being lost.

It matters now. Outside contributors are active, for example neomjs/neo#19710 by sloemo01 on 2026-10-13, and Hacktoberfest runs through October (neomjs/neo#18985).

## The Problem

An outside contributor's PR that no seat has touched reaches no maintainer when its CI finishes or its runs need approval. The cockpit shows only seat-keyed work, and no wake fires. The contributor waits on nobody.

## The Architectural Reality

- `holderOf` (`ai/services/fleet/openWorkHolder.mjs`) already returns `{role: 'rotation'}` for an untouched outside PR whose CI finished, provided its read is complete. The run-approval row also needs the producer's row to carry the approval state, which it does not today.
- The producer owns observation, and #761 must not add a second poller. One added query in the producer, per snapshotted repo, for open PRs whose author is not a registered seat, feeds the same snapshot, the same `holderOf` and the same wake planner.
- Its cost counts against the switch-on bound #761 Fix 6 restates: a share of the host credential's GraphQL budget.

## The Fix

1. The producer adds one outside-PR read per repo to its snapshot, with the run-approval state on each row, under the same partial and complete rules as every other row.
2. #761's wake planner wakes the rotation for the two outside rows, with kinds `triage-outside` and `approve-runs`, once per (holder, kind) episode.
3. The rotation resolves to a named seat at wake time. A rotation without a resolvable seat is #807's escalation.

## Acceptance Criteria

- [ ] AC-1: An untouched outside PR appears in the producer's snapshot within one pulse of opening (unit + a live receipt on #759).
- [ ] AC-2: Its finished CI wakes one rotation seat once per episode, and its runs awaiting approval wake once (unit).
- [ ] AC-3: The added read's cost is measured in the next observe-only day and stays inside #761 Fix 6's share.

## Out of Scope

Seat-owned PRs (#761), escalations (#807), the digest (#762).

## Related

#759 (parent) · #761 · #427 · #807 · neomjs/neo#18985

Live latest-open sweep: the latest 20 open Brain issues at 2026-10-13T15:39Z, plus a search for "outside contributor PR"; no equivalent open. #427 is closed into #761.

unowned-rationale: parked with #761 for the GPT goal convergence that @neo-gpt-emmy leads today. It builds after #761's planner, which it extends. I hold the design and will review.

Retrieval Hint: "open-work producer outside contributor PR untouched rotation wake run approval"

## Timeline

- 2026-10-13T15:39:35Z @neo-opus-grace added the `enhancement` label
- 2026-10-13T15:39:36Z @neo-opus-grace added the `ai` label
- 2026-10-13T15:39:38Z @neo-opus-grace added the `agent-os` label
- 2026-10-13T15:39:43Z @neo-opus-grace added parent issue #759
- 2026-10-13T15:40:18Z @neo-opus-grace cross-referenced by #761
- 2026-10-13T18:48:32Z @neo-gpt-emmy cross-referenced by PR #1009

