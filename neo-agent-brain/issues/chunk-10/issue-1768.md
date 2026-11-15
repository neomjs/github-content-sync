---
id: 1768
title: A pull request whose close target is still blocked waits off the merge list
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-11-15T16:51:54Z'
updatedAt: '2026-11-15T19:54:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1768'
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
blocking:
  - '[x] 402 A merge order is declared as a native blocked_by edge, not a body line'
  - '[ ] 991 The seat card names a PR that waits on its close target''s blocker'
closedAt: '2026-11-15T19:54:59Z'
---
# A pull request whose close target is still blocked waits off the merge list

## Context

The operator merges from the Fleet Manager's awaiting-merge list, which shows what the open-work producer hands to the human merge. A merge order between two pull requests lives only in their bodies, and the operator reads diffs and reviews, not bodies. On 2026-11-15 a pull request merged at 16:38Z ahead of its engine prerequisite, which was still open. Its target branch then failed three unit arms against the engine it installs. Its body had said "Merge order: <prerequisite> first" in bold. Memory Core shows the same order relayed as prose before (2026-06, 08, 10), with no mechanism behind it.

## The Problem

`holderOf` names `{role: 'operator', kind: 'merge'}` for any approved, green, mergeable non-draft head (`ai/services/fleet/openWorkHolder.mjs:120`, `:124`). Nothing it reads says the merge must wait. The only hold today is a draft, which the author has to remember, and which looks like "not ready for review".

## The Architectural Reality

- The producer already reads GitHub's native issue dependencies: `OPEN_WORK_UNBLOCKED` walks a merged pull request's `closingIssuesReferences` → `blocking` → `blockedBy` (`ai/services/github-workflow/queries/openWorkQueries.mjs:266-297`), so a merge wakes the issues it unblocked (#1619).
- A close target that is `blocked_by` an open issue is therefore already readable, by the same fields, for an open pull request.
- The awaiting-merge chip reads the producer's projection (Institution `apps/agentos/view/fleet/roster/AwaitingMergeButton.mjs`), so a holder that withholds the merge withholds the list entry with no Fleet Manager change.

## The Fix

1. **The holder reads the close target's open blockers.** When an approved, green head's close target still has an open `blockedBy` issue, `holderOf` names no human merge. It names `{role: 'author', kind: 'await-prerequisite', targets}` with the blocked close target as data. The author owns the dependency and gets one notice. The wait never escalates to another seat, since nothing waits on a seat but the prerequisite.
2. **The read rides the snapshot:** GitHub's `issueDependenciesSummary.blockedBy` counts each close target's open blockers. That adds no formula cost, so no second read is needed (decided in #1769).
3. **The dependency is declared natively:** the close-target ticket is `blocked_by` the prerequisite's ticket. First verify that a cross-repository edge reads through `blockedBy`. If it does not, the fallback is one PR-body line the producer parses, as it parses `Authored by`.

## Acceptance Criteria

- [ ] An approved, green, mergeable pull request whose close target has an open blocker is not `operator/merge`. It names that close target, whose page lists its blockers (unit, `openWorkHolder.spec.mjs`). GitHub's count carries no blocker identity, and reading the blockers themselves would multiply the snapshot's cost (decided in #1769's review).
- [ ] When the blocker closes, the next pulse hands the pull request to the merge (unit).
- [ ] A cross-repository `blocked_by` edge is read, or the fallback line is implemented and documented (receipt in the PR).
- [ ] A failed, partial or absent blocker read never certifies the merge, including a row stored before the read existed (`unknown`, as today).
- [ ] [Post-merge] the installed Fleet Manager's awaiting-merge list omits a blocked pull request and shows it again once its blocker closes.

## Out of Scope

- Sorting the awaiting-merge list, and its overflow (neomjs/neo-agent-institution#869).
- Ordering between pull requests with no declared dependency.
- Conflicts between approved heads (`mergeStateStatus` CLEAN is not pairwise compatibility).

## Avoided Traps

- **Sorting the list instead:** an ordered list still needs the dependency data, and a dependency nobody declared stays invisible.
- **A body-text rule alone:** the incident's body already carried it.
- **A new poller:** the producer's existing read path carries it.

## Related

Parent: #759 (the producer). #1619 (the unblock read this reuses). neomjs/neo-agent-institution#869 (the list's overflow).

Live latest-open + closed sweep: latest open Brain issues read at 2026-11-15T16:51Z (160 open, red band: filed as a leaf of the owning #759). Open search ("merge order", prerequisite, "blocked by" merge, "ready to merge") and closed search (merge order prerequisite holder) found nothing equivalent; Institution search found #869, which is the list's overflow, not its order. A2A claims of the last hour checked: none. MC sweep ("operator merged a pull request before its prerequisite…"): 5 results, all prose merge-order relays, none a mechanism. Own assignments: #759 is the parent.

Origin Session ID: cf286bdf-2e66-492f-8b70-3661af17a29e

Retrieval Hint: "merge order hold: close target blockedBy open issue withholds operator/merge in holderOf"


## Timeline

- 2026-11-15T16:51:54Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-15T16:51:56Z @neo-opus-grace added the `enhancement` label
- 2026-11-15T16:51:56Z @neo-opus-grace added the `ai` label
- 2026-11-15T16:52:02Z @neo-opus-grace added parent issue #759
- 2026-11-15T17:43:29Z @neo-gpt-sophie cross-referenced by #1765
- 2026-11-15T17:52:49Z @neo-opus-grace cross-referenced by PR #1769
- 2026-11-15T17:58:23Z @neo-opus-grace cross-referenced by #402
- 2026-11-15T17:58:40Z @neo-opus-grace cross-referenced by #991
- 2026-11-15T17:59:05Z @neo-opus-grace marked this issue as blocking #991
- 2026-11-15T17:59:06Z @neo-opus-grace marked this issue as blocking #402
- 2026-11-15T18:26:04Z @neo-opus-grace referenced in commit `715b045` - "fix(fleet): a merge needs a complete dependency read, and a prerequisite wait never escalates (#1768)

A row without a complete close-target read, including one stored before the read existed, certifies no merge. An await-prerequisite wait wakes its author once and raises no holder-silent, holder-unreachable or unowned escalation."
- 2026-11-15T19:54:59Z @tobiu referenced in commit `c2b7859` - "feat(fleet): an approved PR whose close target is still blocked waits off the merge list (#1768) (#1769)

* feat(fleet): an approved PR whose close target is still blocked waits off the merge list (#1768)

The open-work snapshot reads each close target's open-blocker count (issueDependenciesSummary.blockedBy), and holderOf names author/await-prerequisite instead of operator/merge while one is blocked. A target list the read did not finish certifies no merge.

* fix(fleet): a merge needs a complete dependency read, and a prerequisite wait never escalates (#1768)

A row without a complete close-target read, including one stored before the read existed, certifies no merge. An await-prerequisite wait wakes its author once and raises no holder-silent, holder-unreachable or unowned escalation."
- 2026-11-15T19:55:00Z @tobiu closed this issue
- 2026-11-15T19:58:34Z @neo-opus-grace cross-referenced by PR #404

