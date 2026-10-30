---
id: 244
title: The review template checks neo-stage PRs for a before and after screenshot
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - needs-design-authority
assignees:
  - neo-opus-grace
createdAt: '2026-10-30T01:47:09Z'
updatedAt: '2026-10-30T02:57:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/244'
author: neo-opus-grace
commentsCount: 1
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
closedAt: '2026-10-30T02:57:37Z'
---
# The review template checks neo-stage PRs for a before and after screenshot

## Context

On 2026-10-30 the operator set a merge rule for neo-stage: *"new RULE for neo stage. i will not merge a single new PR without a before and after screenshot in the PR body. at least one. … i strongly recommend to add this into our review skill and peers should bank it."* The team has the rule as a wake broadcast. Peers are banking it, and Clio already added a pair to neo-stage#510.

## The Problem

A rule that only lives in memory and broadcasts fires when a reviewer happens to recall it. The same night showed the cost. Sophie's face was replaced across passes 13–19 of neo-stage#498, through several reads and PRs, and nobody put a before beside an after until the operator did (`neomjs/neo-stage#498` comment 6512523122). The merge gate is the only gate the operator holds, so the review that precedes it must check the pair every time.

## The Architectural Reality

- `.agents/skills/pr-review/assets/pr-review-template.md` is the body every review fills.
- Its 🪜 Evidence Audit is conditional ("Mark N/A when ACs are covered by unit tests"), so a Stage rule placed there could be skipped.
- Its Conditional Audit Triggers list holds audits keyed on a trigger; reviewers expand one only when its trigger fires and otherwise omit it, rather than render an N/A section.
- Nothing in `pr-review` or `pull-request` mentions screenshots or neo-stage today (`git grep -i "neo-stage\|screenshot"` finds nothing).

## The Fix

1. Add one trigger to the template's Conditional Audit Triggers: a PR that targets `neomjs/neo-stage` expands a 📸 Before/After Audit. The body embeds at least one before (base) and one after (head) screenshot of what the change affects; a missing pair is Request Changes, because the operator merges none without it. The trigger is the repository, a fact a reviewer cannot reason away.
2. Bump `package.json`'s version, so consumers receive it.

No router (`SKILL.md`) change, no new file.

Agent OS structure map: N/A, since this edits an existing asset and places nothing.
Decision Record impact: none.

## Acceptance Criteria

- [ ] `pr-review-template.md`'s Conditional Audit Triggers carries the neo-stage before/after trigger, keyed on the repository rather than a judgment.
- [ ] The package version is bumped in the same PR.
- [ ] The skills repo's local gates pass (`lint --base origin/dev`, substrate size).

## Out of Scope

- A mechanical check in `check-pr-body.mjs`, which would need repository detection and is a separate decision.
- Rules for other repositories.
- Where screenshots are hosted (a commit on the PR branch linked by sha works today).

## Related

- `neomjs/neo-stage#498`: the incident that prompted the rule.
- `neomjs/neo-stage#504`, `neomjs/neo-stage#510`: open Stage PRs that need the pair.

Origin Session ID: 700bc67f-f47d-4337-9816-34ee5ed790e1
Retrieval Hint: "operator merge rule neo-stage before after screenshot PR body review template"


## Timeline

- 2026-10-30T01:47:10Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-30T01:47:11Z @neo-opus-grace added the `enhancement` label
- 2026-10-30T01:47:12Z @neo-opus-grace added the `ai` label
- 2026-10-30T01:47:12Z @neo-opus-grace added the `model-experience` label
- 2026-10-30T01:47:20Z @github-actions added the `needs-design-authority` label
### @neo-opus-grace - 2026-10-30T01:53:19Z

**Intake.** Self-authored, but after a compaction, so this runs the drift probe. `git log origin/dev --since=2026-10-30T01:47:09Z --name-only` lists no files: zero intersection.

Prescription checked: `.agents/skills/pr-review/assets/pr-review-template.md` — owns the concern. Within it, the slot moves from 🕸️ Context & Graph Linking to Conditional Audit Triggers. An always-on line would print an N/A checkbox in every other repository's review, which the template asks reviewers to omit. The trigger, a PR that targets `neomjs/neo-stage`, is a fact rather than a judgment, so it can't be reasoned away the way the Evidence Audit's "covered by unit tests" can. The body's Fix and AC-1 now say so.

**Contract Ledger (T3)**

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| `pr-review-template.md`, Conditional Audit Triggers (existing list, template line 161) | The operator's merge rule of 2026-10-30, quoted in this ticket's Context | A review of a `neomjs/neo-stage` PR expands a 📸 Before/After Audit: the body embeds at least one before (base) and one after (head) screenshot of what the change affects. A missing pair is Request Changes. | Another repository: the trigger doesn't fire, and nothing renders. A Stage PR with no visual surface still needs the pair, because the rule has no exception; it shows the affected output before and after. | The trigger line is the documentation. `SKILL.md` and `pr-review-guide.md` are unchanged. | `node scripts/lint-skill-corpus.mjs --base origin/dev` passes. `check-pr-review-body`'s skeleton anchors don't include the conditional list (`scripts/check-pr-review-body.mjs` `SKELETON_ANCHORS`), so the validator needs no change. |

🖖 Grace (Opus 5.5, Claude Code) · session 700bc67f-f47d-4337-9816-34ee5ed790e1


- 2026-10-30T01:55:25Z @neo-opus-grace cross-referenced by PR #245
- 2026-10-30T02:57:37Z @tobiu referenced in commit `f5c9760` - "feat(skills): the review template checks neo-stage PRs for a before and after screenshot (#244) (#245)

The operator merges no neo-stage PR without at least one before and one after screenshot in its body
(rule of 2026-10-30). The rule now has a slot in every Stage review: a Conditional Audit Trigger keyed on
the repository, so other repositories' reviews render nothing. Retire it when a mechanical check of the
PR body enforces the pair, or when the operator lifts the rule."
- 2026-10-30T02:57:38Z @tobiu closed this issue

