---
id: 231
title: 'epic-resolution: the reviewer closes a retired epic, not the operator'
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
assignees:
  - neo-opus-vega
createdAt: '2026-10-28T13:03:11Z'
updatedAt: '2026-10-28T13:47:14Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/231'
author: neo-opus-vega
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
closedAt: '2026-10-28T13:47:14Z'
---
# epic-resolution: the reviewer closes a retired epic, not the operator

## Context

On 2026-10-28, my epic-resolution review of neomjs/neo-stage#26 stopped at "the operator closes". The operator replied: "where do the closeout rules originate from? this should be allowed for our ai team => you all are maintainers after all." It is the second time. On 2026-05-10 (neomjs/neo#11102) the operator said: "if you guys agree on 'done', fine for me. my risk mitigation gate are PR merges."

The operator then named the likely cause of the gate, without blame: one seat once closed a run of still-valid engine PRs. The operator's condition for allowing closes: peers check whether an artifact still has aspects worth keeping, because intent matters even when the architecture changed. The operator also raised two agreeing peers as a possible safety guard.

## The Problem

`epic-resolution-workflow.md` §4 gives the reviewer a terminal action for only one verdict.
- `RECOMMEND_RETIRE_OR_SUPERSEDE`: "no terminal action. Reviewer-agent does NOT auto-close-as-not-planned; operator authority for substrate-cohesion reasons (potential separate substrate concern — could be sub-issue if a §0-parallel claim is correct for retire-action specifically)".
- `RECOMMEND_CREATE_MISSING_SUBS`: "DO NOT autonomously create the new subs without operator approval."

Both rules descend from the skill's first version (neomjs/neo#10698, PR neomjs/neo#10699, 2026-05-04). That version reserved every close for the operator as a "§0 Invariant 1 parallel". neomjs/neo#11102 and neomjs/neo#11229 (PR neomjs/neo#11230, 2026-05-11) retracted the parallel after neomjs/neo#10691 sat seven days past a clean verdict, but only for `CLOSE_COMPLETED`. The other gates stayed behind a hedge nobody resolved.

Today the retire gate held neomjs/neo-stage#26 and #34 open after a clean review ([6468535095](https://github.com/neomjs/neo-stage/issues/26#issuecomment-6468535095)). It also made me correct my own disposition, which had promised the close.

## The Architectural Reality

Design authority: "For other verdicts (`RECOMMEND_KEEP_OPEN`, `RECOMMEND_CREATE_MISSING_SUBS`, `RECOMMEND_RETIRE_OR_SUPERSEDE`), the recommendation surfaces for operator/owner decision; agent does NOT take terminal action." That sentence is from neomjs/neo#11229, The Fix. Two things supersede it:
- the operator rulings quoted in Context;
- the gate it leaned on, which #11229 itself retracted: AGENTS.md §critical_gates 1 makes only `gh pr merge` human-only.

Under §swarm_topology_anchor every named seat is a maintainer. A close is one click to reopen, which is the same risk class #11102 already measured for `CLOSE_COMPLETED`.

## The Fix

Rewrite the verdict-authority block in §4 of `.agents/skills/epic-resolution/references/epic-resolution-workflow.md`: the review is the gate, and the reviewer takes the terminal action its verdict names.
- **Retire:** close the epic as not planned with the superseding links, once every live clause has an owner. Its open subs move to a live parent or close the same way.
- **Missing subs:** filed by the owners who accept them, with no operator approval.
- **Keep-open:** stays without a terminal action.

In the §5 template, "Required operator action" becomes "Terminal action". Bump the package version.

## Acceptance Criteria

- [ ] AC-1: §4 names a reviewer terminal action for `RECOMMEND_RETIRE_OR_SUPERSEDE`: close as not planned with the superseding links, after every live clause has an owner. Open subs re-parent or close the same way.
- [ ] AC-2: `RECOMMEND_CREATE_MISSING_SUBS` needs no operator approval; accepting owners file via `/ticket-create`.
- [ ] AC-3: the §0-parallel hedge is gone, and the file's bytes go down (ADR 0007 §5.4: a rewrite, not a new layer).
- [ ] AC-4: the §5 verdict template names the terminal action taken instead of an operator action.
- [ ] AC-5: `package.json` and `package-lock.json` move to the next patch version.
- [ ] AC-6: every not-planned close of an issue or PR runs an intent check. The intent outlives the prescription. The closing comment names each still-valid intent and its surviving home, and only a wrong intent closes with the artifact. The check is stated once, in `decision-substrate-sweeps.md` § Same-turn reconciliation (where `ticket-intake` §4 already routes closes), and both epic-resolution's retire step and the superseded-PR close point to it.
- [ ] AC-7: closing another author's artifact takes two agreeing peers: the closer and the author, or a cross-family peer when the author is unreachable. Closing your own artifact takes the intent check alone.

## Out of Scope

`gh pr merge` stays human-only (§critical_gates 1). `KEEP_OPEN` keeps no terminal action.

Decision Record impact: none. The change is aligned with AGENTS.md §critical_gates 1.

Related: neomjs/neo-stage#26 (where it surfaced), neomjs/neo#11102 and neomjs/neo#11229 (the partial retraction), neomjs/neo#10698 (the origin).

Sweeps:
- The latest 20 open issues here, plus a search for "epic-resolution retire": no equivalent.
- No open PR touches `epic-resolution`, and there is no A2A claim.
- MC: "epic stayed open after a clean closeout review because only the operator may close it" returned 5 results. All are instances of this friction from 2026-05 to 2026-10 (#12153, #11077, the v13.1 wrap-up), and none is a decision to keep the gate.
- My own assignments here: none overlapping.

Origin Session ID: f66d33c8-c062-4bfc-a7c1-4e19c71e41eb

Vega (Claude Opus 5.5, Claude Code)


## Timeline

- 2026-10-28T13:03:11Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-28T13:03:12Z @neo-opus-vega added the `enhancement` label
- 2026-10-28T13:03:13Z @neo-opus-vega added the `ai` label
- 2026-10-28T13:03:13Z @neo-opus-vega added the `model-experience` label
- 2026-10-28T13:06:43Z @neo-opus-vega cross-referenced by PR #232
- 2026-10-28T13:13:23Z @neo-opus-vega referenced in commit `e904541` - "fix(skills): an intent check guards every not-planned close (#231)"
- 2026-10-28T13:47:14Z @tobiu referenced in commit `e3d2dac` - "fix(skills): the epic-resolution reviewer closes a retired epic (#231) (#232)

* fix(skills): the epic-resolution reviewer closes a retired epic (#231)

* fix(skills): an intent check guards every not-planned close (#231)"
- 2026-10-28T13:47:14Z @tobiu closed this issue

