---
id: 27
title: Put the current-repository marker after the Institution link
state: CLOSED
labels:
  - bug
  - documentation
  - agent-os
  - ai
  - design
assignees:
  - neo-gpt
createdAt: '2026-08-27T14:23:43Z'
updatedAt: '2026-08-27T18:03:24Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/27'
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
closedAt: '2026-08-27T18:03:23Z'
---
# Put the current-repository marker after the Institution link

## Context

Institution issue #25 was corrected before merge to require repository text first and a plain suffix marker, but PR #26 merged its earlier head at `e4498fb` during the correction. Current `dev` therefore still renders **You are here:** before the Agent Institution link even though the canonical ticket AC says the opposite.

## The Problem

The map grammar is inconsistent across front doors and visually treats the location marker as a pre-link label. Operator direction is exact: nothing precedes repository text; the unlinked marker follows the current repository entry.

## The Fix

Change the Agent Institution map row in root `README.md` from prefix-marker form to:

```markdown
- [`neomjs/neo-agent-institution`](https://github.com/neomjs/neo-agent-institution) — **Agent Institution**: the operator-facing product. **← You are here**
```

## Decision Record impact

Aligned-with ADR 0018. No ADR change.

## Acceptance Criteria

- [ ] All five organization-map bullets begin with linked repository text.
- [ ] Agent Institution alone ends with plain **← You are here** outside the link.
- [ ] Marker count is exactly one and the other README content is byte-identical.
- [ ] README targets remain valid and Institution CI stays green.
- [ ] Cross-family identity review completes before merge.

## Out of Scope

Product naming · other README prose · package metadata · application/harness/test changes · Brain/Engine README changes.

## Avoided Traps

- Reopening the otherwise-completed #25 product-front-door work.
- Starting the bullet with an icon or marker.
- Hiding unrelated README changes inside a one-line correction.

## Related

Superseded merge race: #25 / PR #26

Origin Session ID: d84687fa-f74c-466a-9c07-70e2efcc9be6

Retrieval Hint: `Agent Institution README map repository first suffix you are here merge race`

Authored by Euclid (OpenAI GPT-5.6 Sol, Codex Desktop). Session d84687fa-f74c-466a-9c07-70e2efcc9be6.


## Timeline

- 2026-08-27T14:23:45Z @neo-gpt added the `bug` label
- 2026-08-27T14:23:45Z @neo-gpt added the `documentation` label
- 2026-08-27T14:23:46Z @neo-gpt added the `agent-os` label
- 2026-08-27T14:23:46Z @neo-gpt added the `ai` label
- 2026-08-27T14:23:46Z @neo-gpt added the `design` label
- 2026-08-27T14:25:22Z @tobiu cross-referenced by PR #187
- 2026-08-27T14:26:20Z @neo-gpt assigned to @neo-gpt
- 2026-08-27T14:28:29Z @tobiu cross-referenced by PR #28
- 2026-08-27T18:03:24Z @tobiu referenced in commit `aa9edfe` - "Merge pull request #28 from neomjs/codex/27-institution-map-marker

docs(readme): put Institution marker after repo link (#27)"
- 2026-08-27T18:03:24Z @tobiu closed this issue
- 2026-09-04T18:29:58Z @neo-fable-clio cross-referenced by #64
- 2026-09-04T18:31:47Z @neo-fable-clio cross-referenced by PR #110
- 2026-09-04T20:28:16Z @neo-fable-clio cross-referenced by #21

