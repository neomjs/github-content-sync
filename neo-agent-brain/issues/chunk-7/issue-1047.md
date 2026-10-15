---
id: 1047
title: A GitLab seat's lane line reads its GitLab issue claims
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-15T11:16:20Z'
updatedAt: '2026-10-15T11:16:20Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1047'
author: neo-opus-vega
commentsCount: 0
parentIssue: 684
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 1046 The roster''s lane line reads claims from a page that never holds them'
blocking: []
---
# A GitLab seat's lane line reads its GitLab issue claims

## Context

The operator asked on 2026-10-15 to file the GitLab follow-ups now. They are low priority, but they should not be lost. This is the Fleet's GitLab parity goal (#684) on the cockpit's read side; the seat side (#710, #712, #727, #729, #755) is closed.

#1046 makes the roster's lane line multi-source and keeps the lane-fact contract forge-neutral (login, forge, issue reference, claimedAt, line, source). Its forge arm reads GitHub only. A GitLab seat therefore shows its forge lane source as `not-wired` (#1046 AC-1c), and its A2A claims count only where the sharing mode admits broadcasts.

## The Problem

On a team that does not run team sharing, a GitLab seat's lane line stays empty even while the seat holds a self-assigned GitLab issue.

## The Architectural Reality

- `ai/services/fleet/fleetPrLaneActivityAdapter.mjs` is "deliberately source-specific". It maps GitHub issue assignees (`issue-activity`) and `[lane-claim]` issue comments (`commentLane`) only, and every `FLEET_COCKPIT_SOURCES` entry is `github-workflow:*` (`src/fleet/contract/cockpit.mjs`).
- The Fleet already knows each seat's forge: agent definitions carry `forge` / `forgeHost`, and `FleetLifecycleService` places `NEO_GITLAB_PAT` for `gitlab` seats.
- `gitlab-workflow` reads GitLab issues (`list_issues`, `get_local_issue_by_id`) against a self-hosted host (`gitlab.hostUrl` ← `NEO_GITLAB_HOST`).

## The Fix

Add a GitLab lane reader. It maps a GitLab seat's issue assignments and `[lane-claim]` issue comments onto #1046's lane-fact contract with `forge: 'gitlab'` and `gitlab-workflow:*` sources, and reads from the seat's own `forgeHost`. #1046's fold takes it with no change.

## Acceptance Criteria

- [ ] AC-1: A fake GitLab issue payload, with an assignment and a `[lane-claim]` comment, becomes that GitLab seat's lane line. Newest wins across sources.
- [ ] AC-2: A GitLab seat's forge lane source reports `wired` from this reader, which replaces #1046's AC-1c `not-wired`.
- [ ] AC-3: A self-hosted host is read from the seat's `forgeHost`, never assumed to be gitlab.com.
- [ ] AC-4: GitHub seats' lane lines are unchanged.

## Out of Scope

- Merge requests and issue activity in the open-work and activity producers: that is the sibling GitLab leaf.

**Priority:** low (operator, 2026-10-15). Blocked by #1046, whose contract this plugs into.

**Sweeps (2026-10-15T11:1xZ):**
- Open-issue search for "gitlab" in Brain, Institution and neo: related #684 (the parity umbrella) and #150 (the MCP server epic); no equivalent.
- A2A: no claim on this scope.
- MC: no prior decision on GitLab cockpit reads.

unowned-rationale: low priority by operator decision. No team runs the Fleet on GitLab yet. Parked under #684.

Origin Session ID: 2efbd8f3-1462-47da-af2e-5ba6501d5d09

Vega (Opus 5.5, Claude Code)


## Timeline

- 2026-10-15T11:16:21Z @neo-opus-vega added the `enhancement` label
- 2026-10-15T11:16:21Z @neo-opus-vega added the `ai` label
- 2026-10-15T11:16:22Z @neo-opus-vega added the `agent-os` label
- 2026-10-15T11:16:34Z @neo-opus-vega added parent issue #684
- 2026-10-15T11:16:37Z @neo-opus-vega marked this issue as being blocked by #1046
- 2026-10-15T11:16:40Z @neo-opus-vega cross-referenced by #1046

