---
id: 439
title: The review premise asks whether the handled state can exist in practice
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-11-23T22:04:54Z'
updatedAt: '2026-11-24T01:09:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/439'
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
closedAt: '2026-11-24T01:09:31Z'
---
# The review premise asks whether the handled state can exist in practice

## Context

neo #22117 (closed unmerged, 2026-11-23) fixed neo #21974 by scanning every parked update owner on the VDom hot path of every update. Its justification was `OwedFlightFlags.spec.mjs`, whose red came from writing `'other-window'` into private `_windowId` fields. That is a parent/child window split the real app never produces: 231/231 real edges agree, with 0 mismatches in 7,089 samples. The author's checks, my reviewer verification and a red-first test all let it through. The operator caught it: *"can this really exist in practice? … the risk is real: adding bloat and performance drains, where it is not needed."*

## The Problem

Red→green proves that a test reacts to a change, not that the defect state occurs. Nothing in a review or a PR body asks for that receipt, so the question fires only when someone happens to think of it.

## The Architectural Reality

- `.agents/skills/pr-review/assets/pr-review-template.md` L29–36: the Patch-Blind Premise Snapshot has Inputs, Expected Solution Shape, Patch Verdict and Premise Coherence. None asks whether the handled state occurs.
- `.agents/skills/pr-review/audits/cycle-1-premise-preflight.md` L9: the Premise-invalid row asks whether the premise is false, but requires no evidence.
- `.agents/skills/pull-request/references/pull-request-workflow.md` §9: the body's Test Evidence slot asks for outside-CI receipts, not for the state's occurrence.

## The Fix

1. Premise snapshot: add one field, **In Practice**: "Can the state this change handles exist in practice? Receipt: a real-run observation, or a real flow that reaches it through public behavior. A setup that writes private fields or patches internals is manufactured and does not count. No receipt ⇒ Premise-invalid fires."
2. The Premise-invalid catalog row gains: "…or the change handles a state with no in-practice receipt".
3. Pull-request §9, Test Evidence: a bug fix names the real flow that is red on its base.

Fold into existing lines where possible, and measure against the corpus byte budgets.

## Acceptance Criteria

- [ ] The review template and the premise catalog carry the In Practice question and its receipt rule.
- [ ] Pull-request §9 asks a bug fix for the real flow that is red on base.
- [ ] `npm run lint` passes the corpus budgets, and a review written from the template still validates.
- [ ] The package version is bumped in the same PR.

## Out of Scope

Test-oracle guidance (#191), class-idiom checks (#432), new audit files.

Decision Record impact: none.

## Related

#191, #432, neomjs/neo#22117, neomjs/neo#21974

Live latest-open + closed sweep: latest open issues + open and closed searches (manufactured, private field, fixture realism, test premise, premise snapshot, "exist in practice") at 22:04Z; 43 open (orange). Nearest owners: #191 (test oracles; specimen posted there) and #432 (class idioms); neither carries this slot. A2A claim sweep: none. Memory Core: only tonight's #22117 record.

Origin Session ID: 0e538ed3-a923-46e7-9960-89b571ecad82
Retrieval Hint: "can this really exist in practice review premise slot manufactured test state #22117"

## Timeline

- 2026-11-23T22:04:54Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-23T22:04:56Z @neo-opus-grace added the `enhancement` label
- 2026-11-23T22:04:57Z @neo-opus-grace added the `ai` label
- 2026-11-23T23:04:44Z @neo-opus-grace cross-referenced by PR #440
- 2026-11-23T23:33:52Z @neo-opus-grace referenced in commit `46073f2` - "feat(pr-review): guide §0 captures In Practice, with an N/A path for PRs that handle no state (#439)

Round 1 (Maryam): §0 still captured four fields, so a reviewer building the snapshot patch-blind met In Practice only at template time. §0 now holds the receipt rule as field 2, the template slot points at it, and a PR that handles no existing state writes "N/A — no handled state".

[skill-growth-justified: +476 B over origin/dev for a rule loaded only when the pr-review and pull-request skills trigger, never per turn; its single home is guide §0 beside the four fields it joins]"
- 2026-11-24T01:09:31Z @tobiu referenced in commit `b6854fd` - "feat(pr-review): the premise snapshot asks whether the handled state can exist in practice (#439) (#440)

* feat(pr-review): the premise snapshot asks whether the handled state can exist in practice (#439)

A red-to-green test proves that a test reacts to a change, not that the state it builds occurs. The Patch-Blind Premise Snapshot gains an In Practice field with its receipt rule, the Premise-invalid catalog row fires on a missing receipt, and the pull-request Test Evidence slot asks a bug fix for its real flow that is red on base. Version 2.5.0.

* feat(pr-review): guide §0 captures In Practice, with an N/A path for PRs that handle no state (#439)

Round 1 (Maryam): §0 still captured four fields, so a reviewer building the snapshot patch-blind met In Practice only at template time. §0 now holds the receipt rule as field 2, the template slot points at it, and a PR that handles no existing state writes "N/A — no handled state".

[skill-growth-justified: +476 B over origin/dev for a rule loaded only when the pr-review and pull-request skills trigger, never per turn; its single home is guide §0 beside the four fields it joins]"
- 2026-11-24T01:09:33Z @tobiu closed this issue

