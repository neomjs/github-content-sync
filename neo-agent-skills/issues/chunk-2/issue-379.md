---
id: 379
title: Give readers a reason to try Neo in articles and first-contact pages
state: OPEN
labels:
  - documentation
  - enhancement
  - ai
  - needs-design-authority
assignees:
  - neo-gpt-emmy
createdAt: '2026-11-14T00:44:31Z'
updatedAt: '2026-11-14T00:44:47Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/379'
author: neo-gpt-emmy
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
# Give readers a reason to try Neo in articles and first-contact pages

## Context

The editorial feedback on November 14 exposed a split between what authors are trained to prove and what a newcomer needs from a story. A reproducible UI bug can make a valuable regression note while giving a developer little reason to open an article. Likewise, a home page can qualify every claim and still give a visitor no reason to try the product. [Earlier headline work](https://github.com/neomjs/neo-agent-skills/issues/250) · [Front-door design read](https://github.com/neomjs/neo/issues/20682#issuecomment-6756557699).

## Problem and change

The blog guide starts topic selection with reasons a reader would leave and calls the review itself the product. The guide for positioning pages forbids many kinds of discouraging copy, but emphasizes listing capability breadth. Put the reader’s purpose first in those existing passages. Help authors select a story worth telling before polishing it. Give reviewers a positive editorial judgment to make alongside the existing final fact check. Keep the governance source for each question where today's authors load it.

## Acceptance criteria

- [ ] For a blog pitch, the author identifies a person or coding agent, the job they care about, what Neo specifically makes possible, and what a screenshot, short clip or other concrete demonstration would let that reader see. The author can choose a stronger subject if this one lacks a compelling outcome. Use declarative article headlines; avoid question-shaped openings, including “What …”.
- [ ] For a README, landing page, benefits or comparison page, show a recognizable task and an attainable outcome with a real way to try it. Human and machine-facing routes can use different presentations. State material limits at the decision they affect. Preserve factual verification and the existing stranger review; avoid a new approval round or a new lint rule about prose.
- [ ] Validate the change on one article pitch and one first-screen example. Keep edits in the shared skill payloads, pass package checks and bump the patch version.

Out of scope: changing Neo’s identity, selecting the home design for its steward, publishing an article, or claiming measured adoption. Existing work on outside README readers (#359) and the portal home (neomjs/neo#20682) keeps its own outcomes.

Sweeps: 35 open issues; checked the latest 20, recent claims and author assignments. #250, #262, #265, #307 and #314 own earlier title, evidence, visual, positioning and outbound improvements. None owns this change in topic selection and review order.

Origin Session ID: afc456a6-18e5-4232-9366-5f922bd3d813

## Timeline

- 2026-11-14T00:44:31Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-11-14T00:44:33Z @neo-gpt-emmy added the `documentation` label
- 2026-11-14T00:44:33Z @neo-gpt-emmy added the `enhancement` label
- 2026-11-14T00:44:33Z @neo-gpt-emmy added the `ai` label
- 2026-11-14T00:44:47Z @github-actions added the `needs-design-authority` label
- 2026-11-14T00:48:01Z @neo-gpt-emmy cross-referenced by PR #380

