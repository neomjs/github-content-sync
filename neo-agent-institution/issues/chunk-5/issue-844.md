---
id: 844
title: A plane outage shorter than the cockpit's refresh is never named
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-30T22:44:46Z'
updatedAt: '2026-10-31T22:58:05Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/844'
author: neo-opus-ada
commentsCount: 0
parentIssue: 424
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-31T22:58:05Z'
---
# A plane outage shorter than the cockpit's refresh is never named

## Context

In row 5's installed walk on 2026-10-30 ([#516 receipt 6534998715](https://github.com/neomjs/neo-agent-institution/issues/516#issuecomment-6534998715)), H2 re-cut the plane (`cut.sh`, same revision `4ca0f506`):

- mc/kb/fleet-server were recreated 21:59:03–21:59:14Z, the orchestrator at 21:59:37Z, and all four were healthy at 21:59:49Z.
- Neural Link reads of the cockpit at 21:59:19Z, 21:59:42Z and ~22:00:20Z all showed the unchanged pre-outage state ("feed partial").
- The spine never named the outage. For comparison, H1's 5.5-minute stop was named "Fleet feed degraded — showing last-known data".

## The Problem

A plane outage shorter than the cockpit's read cadence passes without a trace. Seats lost Memory Core during the recreate, while the cockpit implied nothing had happened. The step list (#516, 6457088113) expects H2 to read "as in H1 during the recreate".

## The Architectural Reality

The spine derives from the cockpit's most recent fleet/roster/activity reads. These are polled (`FleetAdmission` `DEFAULT_ACTIVITY_RECHECK_MS` = 60 000), so a failed or refused read inside a short window leaves no state behind once the next read succeeds. The plane-side deployment state (`get_deployment_state_snapshot`) records service restarts independently of the cockpit's polling.

*(Corrected 2026-10-31, in review of PR #854.)* The cockpit never reads that snapshot. It reads `fleetDeploymentState`, whose per-service projection keeps no start time, so a restart between two of its reads (every 120 s) leaves nothing on the cockpit's wire. neomjs/neo-agent-brain#1409 adds `services[].startedAt`.

## The Fix

Name a recent outage even when it ends between reads. An outage that ends between two reads leaves no failed read behind, so the restart route below is the one AC-1 needs; the failed-read route also names an outage that a read did observe.
- the cockpit keeps the last failed read or transport error (time + reason) and shows a dismissible "Plane restarted HH:MM–HH:MM — recovered" note once it recovers;
- the spine reads each service's start time from `fleetDeploymentState` (`services[].startedAt`, neomjs/neo-agent-brain#1409) and names any restart newer than the previous read.

The restart route is the one that names a plane cut (AC-3).

## Acceptance Criteria

- [ ] AC-1 A plane outage of ≤ 60 s between two cockpit reads produces a spine note, with its reason and time window, after recovery (spec with a fake clock or fake transport).
- [ ] AC-2 Recovery stays automatic: the note needs no action and does not block `live`.
- [ ] AC-3 (installed) Row 5's H2 re-walk receipt shows the note for a plane cut.

## Out of Scope

Long outages (already named, H1) and the cold-launch roster timeout (#841).

## Related

#424, #516, #843. Blocked by neomjs/neo-agent-brain#1409 (the start time on the wire).

Live latest-open + closed sweep: checked the latest 20 open issues plus a search ("outage spine short") at 2026-10-30T22:5xZ; no equivalent. A2A sweep (last 15, through 22:25Z): no claim on this scope.

unowned-rationale: Ready for an FM maintainer; Ada stewards row 5 and takes it if unclaimed.

Retrieval Hint: "cockpit spine short plane outage between reads never named H2"

Origin Session ID: 2e07d6d3-af03-4b39-9254-f979cc291d2f


## Timeline

- 2026-10-30T22:44:48Z @neo-opus-ada added the `bug` label
- 2026-10-30T22:44:48Z @neo-opus-ada added the `agent-os` label
- 2026-10-30T22:44:48Z @neo-opus-ada added the `ai` label
- 2026-10-30T22:45:39Z @neo-opus-ada added parent issue #424
- 2026-10-31T20:36:02Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-31T20:54:52Z @neo-gemini-maryam cross-referenced by PR #854
- 2026-10-31T21:00:15Z @neo-gemini-maryam referenced in commit `3ad4aec` - "chore(test): stamp visual baseline inputs for #844 (#844)"
- 2026-10-31T21:20:34Z @neo-opus-ada cross-referenced by #1409
- 2026-10-31T21:30:40Z @neo-opus-ada cross-referenced by PR #1413
- 2026-10-31T22:03:08Z @neo-gemini-maryam referenced in commit `fc8d7f2` - "fix(agentos): address PR #854 review feedback on plane outage note (#844)"
- 2026-10-31T22:29:21Z @neo-gemini-maryam referenced in commit `e5693aa` - "fix(agentos): expire and clear plane outage note on partial feed (#844)"
- 2026-10-31T22:58:05Z @tobiu referenced in commit `ec62d81` - "fix(agentos): name recovered <= 60 s plane outages on the spine (#844) (#854)

* fix(agentos): name recovered <= 60 s plane outages on the spine (#844)

* chore(test): stamp visual baseline inputs for #844 (#844)

* fix(agentos): address PR #854 review feedback on plane outage note (#844)

* fix(agentos): expire and clear plane outage note on partial feed (#844)"
- 2026-10-31T22:58:06Z @tobiu closed this issue

