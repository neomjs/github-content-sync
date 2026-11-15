---
id: 390
title: Prioritize actionable outside pull requests before new internal lanes
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-11-14T22:29:26Z'
updatedAt: '2026-11-15T16:02:22Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/390'
author: neo-gpt-emmy
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
closedAt: '2026-11-14T23:00:02Z'
---
# Prioritize actionable outside pull requests before new internal lanes

Design authority: Operator direction, 2026-11-14: “let’s get the external PR check into post-review-pickup so each peer does it before choosing a new internal lane.”

## Problem
The pickup gate prefers a new author lane before reviewing. An older outside contribution can wait even when a maintainer holds its next action. Open-work routing identifies holders, but a routed wake is not itself a pickup decision.

## Outcome
Before choosing a new internal lane, inspect the existing open-work view for older outside PRs and advance an actionable maintainer-held item. Respect current holder, CI and contributor-owned decisions; use safe maintainer-owned base maintenance when appropriate, and hand approved merge-ready work to the human merge gate. Do not impose repeated contributor rebases or add routine notifications.

## Acceptance criteria
- Pickup directs maintainers to check outside PRs and prioritize older actionable items before a new internal lane, without displacing urgent operator direction or existing lifecycle obligations.
- The guidance separates maintainer, contributor, CI and operator holds, protects author ownership of substantive decisions, and permits safe base-churn maintenance without demanding repeated contributor rebases.
- Guidance uses existing open-work and lifecycle paths, without new routine notifications.

## Contract Ledger
| Consumer | Surface | Contract |
| --- | --- | --- |
| Maintainer at pickup | `.agents/skills/post-review-pickup/references/pre-review-intake-lane-gate.md` | Evaluate existing external PR holder and next action before selecting new internal work. |
| Contributor | Existing PR | Retains substantive design choices; no repeated rebase demand for maintainer-owned base churn. |
| Human operator | Existing merge gate | Receives approved merge-ready work; agents do not merge. |

## Evidence
The existing lane gate prefers a claimable author lane; Brain open-work already exposes outside-PR holder/rotation routing. The change is a pickup decision, not a new routing service.


## Timeline

- 2026-11-14T22:29:26Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-11-14T22:29:28Z @neo-gpt-emmy added the `enhancement` label
- 2026-11-14T22:29:29Z @neo-gpt-emmy added the `ai` label
- 2026-11-14T22:29:29Z @neo-gpt-emmy added the `model-experience` label
- 2026-11-14T22:29:29Z @neo-gpt-emmy added the `agent-os` label
- 2026-11-14T22:30:17Z @github-actions added the `needs-design-authority` label
- 2026-11-14T22:31:33Z @neo-gpt-emmy cross-referenced by PR #391
- 2026-11-14T22:33:12Z @neo-gpt-emmy referenced in commit `f8b5fb1` - "fix(pickup): advance the Skills package version (#390)"
- 2026-11-14T22:46:31Z @neo-gpt-emmy referenced in commit `ddeaa33` - "feat(pickup): prioritize actionable outside contributions (#390)

[skill-growth-justified: Conditional pickup guidance replaces the prior author-first ordering; retire the manual check when the open-work pickup surface enforces and proves this ordering.]"
- 2026-11-14T22:46:31Z @neo-gpt-emmy referenced in commit `1d1c434` - "fix(pickup): advance the Skills package version (#390)"
- 2026-11-14T22:47:12Z @neo-gpt-emmy removed the `needs-design-authority` label
- 2026-11-14T22:47:38Z @neo-gpt-emmy referenced in commit `7e01dce` - "fix(pickup): discover outside PRs and handle fork refusal (#390)"
- 2026-11-14T23:00:02Z @tobiu referenced in commit `23d5afd` - "feat(pickup): prioritize actionable outside contributions (#390) (#391)

* feat(pickup): prioritize actionable outside contributions (#390)

[skill-growth-justified: Conditional pickup guidance replaces the prior author-first ordering; retire the manual check when the open-work pickup surface enforces and proves this ordering.]

* fix(pickup): advance the Skills package version (#390)

* fix(pickup): discover outside PRs and handle fork refusal (#390)"
- 2026-11-14T23:00:03Z @tobiu closed this issue
- 2026-11-15T14:02:59Z @neo-gpt-sophie cross-referenced by #398
- 2026-11-15T14:09:57Z @neo-gpt-sophie cross-referenced by #399
### @neo-gpt-sophie - 2026-11-15T16:02:22Z

The accepted contributor-assistance design in [neo Discussion #21005](https://github.com/orgs/neomjs/discussions/21005) is being transferred under [Skills #398](https://github.com/neomjs/neo-agent-skills/issues/398). [Policy #399](https://github.com/neomjs/neo-agent-skills/issues/399) preserves outside-work priority and separate holds while replacing the refusal/rebase handback coherently; [Brain #1766](https://github.com/neomjs/neo-agent-brain/issues/1766) supplies the actual outside integration/classification route. This does not reopen or erase this completed priority decision, and no new operational policy is claimed active.


