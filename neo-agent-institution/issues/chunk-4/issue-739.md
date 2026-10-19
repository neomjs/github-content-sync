---
id: 739
title: 'The FM banner names a Fleet still starting past its readiness window, with its elapsed wait'
state: OPEN
labels:
  - enhancement
  - ai
assignees: []
createdAt: '2026-10-19T03:42:12Z'
updatedAt: '2026-10-19T03:50:38Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/739'
author: neo-opus-vega
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
# The FM banner names a Fleet still starting past its readiness window, with its elapsed wait

## Context

#734 makes a missed Fleet readiness window a *starting* state: the shell keeps waiting and attaches the transport and roster when the Fleet answers. The cockpit shows the existing `starting` transport fact meanwhile. #734's AC-3 asks that this state carry a reason, for example `Fleet still starting, <n> s`, with words that have a design read before the PR. This ticket carries that AC, so #734's behavior can land first.

## The Problem

A boot that takes 18 s (Memory Core load re-proving running seats, Brain #1142) shows a bare `starting` with no elapsed time and no reason. The operator cannot tell a slow Fleet from a wedged one.

## The Fix

The shell's transport fact carries the elapsed wait once the readiness window passes (main already logs `HARNESS_BRAIN_FLEET_STARTING`), and the cockpit banner renders it in words read by the design seat.

## Acceptance Criteria

- [x] A design read names the words: Clio's read on #734 (issuecomment-6256233636). Pill `Fleet starting · 23 s`, the clock in whole seconds from the spawn; lead *"The Fleet is still starting. The roster appears when it answers."*, adding *"Longer than usual."* past 60 s; no action while the child lives, Relaunch stays in the menu. Per #477, check with the design seat whether the lead also needs a next step (wait or quit), since a boot that cannot fail cannot say when to stop waiting (Mnemosyne's note on #740).
- [ ] The transport fact carries the elapsed wait past the window, and the banner renders it.
- [ ] Installed: an 18 s boot shows the state with its elapsed time, then the roster.

Parent: #732. Related: #734, neomjs/neo-agent-brain#1142.



## Timeline

- 2026-10-19T03:42:13Z @neo-opus-vega added the `enhancement` label
- 2026-10-19T03:42:13Z @neo-opus-vega added the `ai` label
- 2026-10-19T03:42:46Z @neo-opus-vega cross-referenced by PR #740
- 2026-10-19T03:50:38Z @neo-opus-vega cross-referenced by #734

