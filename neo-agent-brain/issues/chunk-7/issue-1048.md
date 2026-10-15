---
id: 1048
title: A GitLab seat's merge requests and issues reach open work and activity
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-15T11:16:22Z'
updatedAt: '2026-10-15T11:38:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1048'
author: neo-opus-vega
commentsCount: 1
parentIssue: 684
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
# A GitLab seat's merge requests and issues reach open work and activity

## Context

The operator asked on 2026-10-15 to file the GitLab follow-ups now. They are low priority, but they should not be lost. This is the Fleet's GitLab parity goal (#684) on the cockpit's read side; the seat side (#710, #712, #727, #729, #755) is closed and GitLab seats can be defined and started.

## The Problem

A GitLab seat's work never reaches the cockpit. Its card shows no open-work badge (the "1 PR · review due" / "red" row on a GitHub seat), and the activity stream has no merge-request or issue events for it. On a GitLab team the roster would look idle while the seats work.

## The Architectural Reality

- **Open work.** `ai/services/fleet/openWorkProducer.mjs` builds rows from GitHub search through `ai/services/github-workflow/queries/openWorkQueries.mjs` (`OPEN_WORK_SNAPSHOT`, `OPEN_WORK_TERMINAL`). `openWorkReducer.mjs` normalizes pull requests into the row kinds the card renders.
- **Activity.** `ai/services/fleet/fleetPrLaneActivityAdapter.mjs` maps GitHub pull requests and issues only. All `FLEET_COCKPIT_SOURCES` are `github-workflow:*`.
- **The forge is already known.** Agent definitions carry `forge` / `forgeHost`.
- **GitLab reads exist.** `gitlab-workflow` reads merge requests (`list_merge_requests`, `get_merge_request`) and issues against a self-hosted host.

## The Fix

Add a GitLab arm to both producers, selected by each seat's `forge`:
- **Open work:** merge requests as open-work rows with the reducer's existing kinds and roles (author or reviewer; red, review-due or approved from pipeline state and approvals).
- **Activity:** merge-request and issue events under `gitlab-workflow:*` sources.

The reducer's row contract and the card stay forge-neutral. The owner may split open work and activity into two PRs at intake.

## Acceptance Criteria

- [ ] AC-1: A GitLab seat's open merge requests appear as open-work rows on its card, with the same kinds and roles as GitHub pull requests.
- [ ] AC-2: GitLab merge-request and issue events appear in the activity stream under `gitlab-workflow:*` sources.
- [ ] AC-3: A self-hosted host is read from the seat's `forgeHost`.
- [ ] AC-4: GitHub seats' rows and events are unchanged.

## Out of Scope

- The roster lane line: that is the sibling GitLab lane-reader leaf, behind #1046.

**Priority:** low (operator, 2026-10-15).

**Sweeps (2026-10-15T11:1xZ):**
- Open-issue search for "gitlab" in Brain, Institution and neo: related #684 and #150; no equivalent.
- A2A: no claim on this scope.
- MC: no prior decision.

unowned-rationale: low priority by operator decision. No team runs the Fleet on GitLab yet. Parked under #684.

Origin Session ID: 2efbd8f3-1462-47da-af2e-5ba6501d5d09

Vega (Opus 5.5, Claude Code)


## Timeline

- 2026-10-15T11:16:22Z @neo-opus-vega added the `enhancement` label
- 2026-10-15T11:16:23Z @neo-opus-vega added the `ai` label
- 2026-10-15T11:16:23Z @neo-opus-vega added the `agent-os` label
- 2026-10-15T11:16:36Z @neo-opus-vega added parent issue #684
- 2026-10-15T11:16:40Z @neo-opus-vega cross-referenced by #1046
### @neo-opus-vega - 2026-10-15T11:38:19Z

**Consumer to update when GitLab rows exist.** Stage's FM handoff (neomjs/neo-stage#83, approved today) builds the source link as `https://github.com/${repo}/pull/${number}` in `ReplayStateProvider.applyRecordContext`. Its replay admission also accepts only github.com URLs.

That is correct while open work is GitHub-only. Once this ticket hands Stage GitLab merge-request rows, either the row carries its own `url` and Stage uses it, or Stage derives the link from the row's forge.

— Vega (Opus 5.5, Claude Code) 🌿



