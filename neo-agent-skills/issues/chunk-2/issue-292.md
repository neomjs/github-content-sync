---
id: 292
title: One-shot logic files its retirement ticket before the review request
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - tech-debt
  - needs-design-authority
assignees:
  - neo-opus-grace
createdAt: '2026-11-06T10:48:17Z'
updatedAt: '2026-11-06T10:48:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/292'
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
---
# One-shot logic files its retirement ticket before the review request

## Context

@tobiu raised this on neomjs/neo-agent-brain#1502 on 2026-11-06. Verbatim: "there MUST be a task to USE the logic, and then retire it." He proposed this workflow: "for these cases create a retire the code ticket FIRST after creating the PR, before asking for a review. the new ticket must have the pre-requisite to update our containers if needed and execute the logic." On where it lives, also verbatim: "a lifetime line for every PR feels over the top, since 99%+ don't need it."

The recurrence, measured on neomjs/neo-agent-brain `origin/dev` the same day:
- 17 of the 18 `ai/scripts/migrations/` modules (7,321 lines) have no production importer and no npm script. The count comes from a static-import grep with a positive control.
- `retireLegacyConversationRows.mjs` stayed after its one run (neomjs/neo-agent-brain#417, closed 2026-10-12) and was still edited by #1369.
- The ADR identity migration (2,175 lines, merged 2026-10-21) has no live-run receipt on neomjs/neo-agent-brain#1010.
- neomjs/neo-agent-brain#1502 adds 2,138 lines for one run on one plane. It was approved twice before anyone asked who runs it after that run.

## The Problem

- **The rule never asks for an owner.** The Engine `AGENTS.md` Accretion Defense requires a retirement trigger for substrate only. For source code it asks to "name the deletion or why there is none". No ticket ever owns that deletion.
- **A promise in a merged PR is lost.** Nobody reads a merged PR's thread.
- **An unowned retirement ticket rots too.** neomjs/neo-agent-brain#191 has named this debt since 2026-08-27 and has no assignee.

## The Architectural Reality

- `pull-request/references/pull-request-workflow.md` §6.2 routes the review request after current-head CI is green. That is the last author step before review, and the point where the operator wants the ticket created.
- `pr-review/references/pr-review-guide.md` §0 field 2 ("Expected solution-shape") is read on every Cycle-1 review.
  - Placing the check here needs no new section.
  - The cycle-1 premise catalog was rejected as the home: its rows default to Drop+Supersede, which is far too heavy for a missing ticket.
- The pr-review loaded surface has a combined budget of 41,357 bytes (`COMBINED_BUDGETS`, `scripts/lint-skill-corpus.mjs`). It currently holds 34,632.

## The Fix

**Author side,** `pull-request-workflow.md` §6.2, before the review request. A PR that adds one-shot logic files its retirement ticket first. One-shot logic means a data fix, migration or repair whose job is done after one run on a deployment. The ticket:
- is assigned to the PR author;
- is natively `blocked_by` the PR's close target, so it turns ready when the PR merges;
- carries these ACs: update the deployment that runs it if needed, run it with a receipt, and delete it together with its tests and any code only it uses.

The PR links it as `Related: #N`.

**Reviewer side,** `pr-review-guide.md` §0 field 2: one clause. One-shot logic without a linked retirement ticket is one Required Action.

The package version is bumped. There is no PR-template line and no lint: the rule fires only for the rare PR it concerns.

Decision Record impact: none.

## Acceptance Criteria

- [ ] **AC-1:** `pull-request-workflow.md` §6.2 carries the author rule: the retirement ticket comes before the review request, is assigned to the author, is `blocked_by` the close target, and has run and delete ACs.
- [ ] **AC-2:** `pr-review-guide.md` §0 field 2 carries the reviewer clause: no linked retirement ticket gives one Required Action.
- [ ] **AC-3:** `npm run lint` passes, including the pr-review combined surface budget.
- [ ] **AC-4:** the package version is bumped.
- [ ] **AC-5:** the PR body carries the §1.1 slot rationale: disposition, trigger frequency × failure severity × enforceability, and the clause's own retirement trigger.

## Out of Scope

- Retiring the one-shot code already in neomjs/neo-agent-brain. That belongs to a leaf under neomjs/neo-agent-brain#191.
- The retirement ticket for neomjs/neo-agent-brain#1502. That one is owed by its author.
- A PR-template line or lint, which the operator rejected.

## Related

neomjs/neo-agent-brain#1502, neomjs/neo-agent-brain#191, neomjs/neo-agent-brain#1237, neomjs/neo-agent-brain#1010.

Live latest-open + closed sweep, 2026-11-06 ~10:50Z: latest 20 open issues, plus a closed search for "one-shot OR retirement OR retire ticket OR migration". No equivalent was found. The nearest sibling is #88 (retiring a stale npm override, a different surface).
- A2A: no claim on this scope in the last 30 messages.
- Memory Core: no prior decision on this rule.
- Own open assignments: #289, #287 and #258, none on this surface.

Ownership: self-assigned, starting now.

Origin Session ID: a33fc3c4-7e98-4300-985d-25a10e216b50
Retrieval Hint: "one-shot logic retirement ticket before review request blocked_by close target"

🖖 Grace (Opus 5.5, Claude Code)

## Timeline

- 2026-11-06T10:48:17Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-06T10:48:19Z @neo-opus-grace added the `enhancement` label
- 2026-11-06T10:48:19Z @neo-opus-grace added the `ai` label
- 2026-11-06T10:48:20Z @neo-opus-grace added the `model-experience` label
- 2026-11-06T10:48:20Z @neo-opus-grace added the `tech-debt` label
- 2026-11-06T10:48:29Z @github-actions added the `needs-design-authority` label
- 2026-11-06T10:51:40Z @neo-opus-grace cross-referenced by PR #293
- 2026-11-06T10:53:31Z @neo-opus-grace cross-referenced by #1506

