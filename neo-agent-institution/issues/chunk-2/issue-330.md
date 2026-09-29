---
id: 330
title: Restore the populated cockpit image in the README
state: CLOSED
labels:
  - bug
  - documentation
  - ai
assignees:
  - neo-gpt
createdAt: '2026-09-29T19:12:16Z'
updatedAt: '2026-09-29T19:27:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/330'
author: neo-gpt
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
closedAt: '2026-09-29T19:27:30Z'
---
# Restore the populated cockpit image in the README

## Context
The operator reported that the README's populated cockpit image was replaced by an empty app. Institution #323 introduced the replacement in commit f1b45041a2d77ed6a08ac26fc4aeadba3d38597f.

## The Problem
The README now leads with cockpit-cold.png, showing zero agents and an unanswered activity stream. That cold-state regression fixture does not communicate the product's agent cards, lifecycle controls or activity stream.

## The Architectural Reality
The populated cockpit-default-shell.png remains tracked in the same visual-snapshot directory. It is an illustrative test scenario, not a capture of today's live fleet. Runtime sample-data removal under #237 does not require an empty marketing image.

## The Fix
Restore the populated image reference and describe the visual scenario accurately in its alt text and caption.

## Acceptance Criteria
- [ ] The README hero shows the existing populated cockpit with agent cards and activity events.
- [ ] Its caption identifies the visual scenario without claiming current live data.
- [ ] The referenced image exists; the diff is limited to the README image, alt text and caption.

## Out of Scope
Runtime data, screenshots, guides and product branding.

## Related
Refs #178, #237, #323.

Live latest-open sweep: latest 20 Institution issues read on 2026-09-29 before filing; no equivalent. A2A sweep: latest 30 messages across read states, no image claim. KB ticket sweep: README screenshot cockpit empty image, five ranked references, no equivalent. Own-assignment sweep: zero open Institution issues. MC sweep: README empty cockpit default agent cards activity stream image marketing, four results, no prior decision found.

Origin Session ID: 01a0ee37-7eaa-7d52-9869-ba5d0de51b43
Retrieval Hint: Institution README empty hero cockpit-default-shell marketing image.


## Timeline

- 2026-09-29T19:12:17Z @neo-gpt assigned to @neo-gpt
- 2026-09-29T19:12:19Z @neo-gpt added the `bug` label
- 2026-09-29T19:12:19Z @neo-gpt added the `documentation` label
- 2026-09-29T19:12:19Z @neo-gpt added the `ai` label
- 2026-09-29T19:13:49Z @neo-gpt cross-referenced by PR #331
- 2026-09-29T19:27:30Z @tobiu referenced in commit `244ffca` - "Merge pull request #331 from neomjs/codex/330-populated-readme-image

docs(readme): restore the populated cockpit image (#330)"
- 2026-09-29T19:27:30Z @tobiu closed this issue

