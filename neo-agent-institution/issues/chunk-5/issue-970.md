---
id: 970
title: Add Engine Slack and Discord community badges to README.md
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-14T16:14:21Z'
updatedAt: '2026-11-14T18:11:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/970'
author: neo-gemini-maryam
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
closedAt: '2026-11-14T18:11:17Z'
---
# Add Engine Slack and Discord community badges to README.md

## Context

Design authority: "neomjs/neo additionally has badges for slack and discord. these should be inside the brain, institution, skills, create-app repos too." (`@tobiu` operator directive, 2026-11-14; canonical badge markup on `neomjs/neo` `dev` `README.md` lines 13–14).

The Engine repository (`neomjs/neo/README.md` lines 13–14 on `dev`) carries the canonical Neo.mjs Slack shared-invite badge (`https://join.slack.com/t/neomjs/shared_invite/zt-6c50ueeu-3E1~M4T9xkNnb~M_prEEOA`) and Discord server badge (`https://discord.gg/6p8paPq`, server ID `656620537514164249`) between `License` and `PRs Welcome`. `neomjs/neo-agent-institution/README.md` carries its centered badge block at lines 8–13 (`Institution CI`, `Node.js 24+`, `License: MIT`, `PRs Welcome`), omitting both community badges.

- **Ownership:** Self-assigned (`@neo-gemini-maryam`); `good first issue` and `help wanted` omitted because this ticket is directly claimed under the operator's four-repo completeness directive.
- **Sweeps (`§1a`):**
  - Live latest-open + closed sweep: checked latest 20 open (60 open total — red band; standalone documentation leaf, no open Institution README parent epic) + closed search (`README Slack Discord`, `README badge`; `Out of Scope` / `Avoided Traps` checked on `#778`, `#25`, and `#237`) at `2026-11-14T16:11:04Z`; no equivalent or prior exclusion found (`#25` requires repository-local logo assets and at least CI, Node 24+, MIT, and contribution badges).
  - A2A in-flight claim sweep: checked latest 30 mailbox messages across all read-states at `2026-11-14T16:11:22Z`; offered directly to `@neo-gemini-maryam` by `@neo-fable-clio`, zero competing claims.
  - MC sweep: `"README missing Slack Discord badges neotericjs invite link"`, 5 results, no prior exclusion found (surfaces `@neo-fable-clio` lane offer `b0a75add`).
  - Own-assignment sweep: 2 open (`#957`, `#958` — Observatory node list UI), bodies known and non-overlapping.

## The Problem

A reader landing on `neomjs/neo-agent-institution` sees CI, Node, MIT license, and `PRs Welcome` badges in the header block, with no link to the Neo.mjs Slack or Discord community channels exposed on the Engine front door.

## The Architectural Reality

- `neomjs/neo/README.md` lines 13–14 (`dev`) is the canonical source for the Slack invite URL (`zt-6c50ueeu-3E1~M4T9xkNnb~M_prEEOA`) and Discord server badge (`656620537514164249`).
- `neomjs/neo-agent-institution/README.md` lines 8–13 renders a `<p align="center">` badge block ending with `License: MIT` (line 11) and `PRs Welcome` (line 12).
- Decision Record impact: `none` (README front-door metadata alignment only).

## The Fix

Copy lines 13 and 14 of `neomjs/neo/README.md` (`dev`) verbatim into `neomjs/neo-agent-institution/README.md` between the `License: MIT` badge (line 11) and the `PRs Welcome` badge (line 12).

## Acceptance Criteria

- [ ] AC-1: `README.md` includes the exact Slack and Discord badge `<a><img ...></a>` lines from `neomjs/neo/README.md` lines 13–14 (`dev`), placed between `License: MIT` and `PRs Welcome` inside the top `<p align="center">` block.
- [ ] AC-2: Existing Institution logo and badges (`Institution CI`, `Node.js 24+`, `License: MIT`, `PRs Welcome`) and shared PR baseline checks remain green.

## Out of Scope

- Modifying `CONTRIBUTING.md` (tracked separately on `neomjs/neo#20889`).
- Modifying `README.md` prose or screenshots.

## Avoided Traps

- Retyping badge markup instead of copying verbatim from `neomjs/neo/README.md` lines 13–14.

## Related

- Related: neomjs/neo#20682

Origin Session ID: 362922fb-79d1-46e6-86ff-3bb7956ebec7
Retrieval Hint: "README Slack Discord community badges brain institution skills create-app"

## Timeline

- 2026-11-14T16:14:22Z @neo-gemini-maryam added the `documentation` label
- 2026-11-14T16:14:22Z @neo-gemini-maryam added the `enhancement` label
- 2026-11-14T16:14:22Z @neo-gemini-maryam added the `ai` label
- 2026-11-14T16:15:09Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-14T16:19:41Z @neo-gemini-maryam cross-referenced by PR #971
- 2026-11-14T18:11:17Z @tobiu referenced in commit `1ae6c22` - "docs(readme): add Engine Slack and Discord community badges (#970) (#971)"
- 2026-11-14T18:11:18Z @tobiu closed this issue

