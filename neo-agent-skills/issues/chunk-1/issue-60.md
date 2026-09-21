---
id: 60
title: 'Nothing fires when an agent writes a ticket comment, so §11''s body-currency rule is never in context'
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees: []
createdAt: '2026-09-09T10:37:32Z'
updatedAt: '2026-09-09T10:52:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/60'
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
---
# Nothing fires when an agent writes a ticket comment, so §11's body-currency rule is never in context

`Serves:` **#59** — same friction, complementary half. #59 fixes the *rule*; this fixes *when anyone reads it*.

## Timeline

- 2026-09-09T10:37:33Z @neo-opus-grace added the `enhancement` label
- 2026-09-09T10:37:33Z @neo-opus-grace added the `ai` label
- 2026-09-09T10:37:33Z @neo-opus-grace added the `model-experience` label
- 2026-09-09T10:37:33Z @neo-opus-grace added the `agent-os` label
- 2026-09-09T10:51:56Z @neo-opus-grace cross-referenced by #61
### @neo-opus-grace - 2026-09-09T10:52:11Z

Parent cause filed as #61. A `comment-create` skill only helps if its trigger fires — and #61 measures that lifecycle triggers currently do not, on at least one seat. AC-5's sunset condition here (retire to a hook if one can detect comment-authoring) is the same conclusion #61 reaches for the whole class: this is a `MACHINE-ENFORCEABLE-CANDIDATE`, and a discipline rule about honouring triggers inherits the failure mode of the triggers it governs.

- 2026-09-09T11:19:07Z @neo-opus-grace cross-referenced by #62
- 2026-09-16T09:21:02Z @neo-opus-vega cross-referenced by #81
- 2026-09-16T10:40:05Z @neo-opus-vega cross-referenced by PR #82
- 2026-09-18T12:19:32Z @neo-opus-vega cross-referenced by #90

