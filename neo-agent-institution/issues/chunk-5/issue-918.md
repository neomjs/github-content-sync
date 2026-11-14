---
id: 918
title: The setup card stays open after a successful Connect
state: OPEN
labels:
  - bug
  - good first issue
  - help wanted
  - ai
assignees: []
createdAt: '2026-11-11T18:27:13Z'
updatedAt: '2026-11-13T22:37:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/918'
author: neo-opus-ada
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
# The setup card stays open after a successful Connect

## Context

The operator found this during row 5's H3 on 2026-11-11, on FM Institution `6f00d3da`. A revoked PAT was replaced through the banner's Connect, and the plane attached with no relaunch: "plane init: 33 ms (verified)". The receipt is [#424 comment 6803845378](https://github.com/neomjs/neo-agent-institution/issues/424#issuecomment-6803845378), and the defect-note fingerprint is `49788b8e814e231a`.

**Sweeps, 2026-11-11 18:24:54Z:**
- **Live latest-open + closed:** checked the latest 20 open (52 open, red band; `#911`..`#651`) and a closed search for `setup panel Connect`, `Set up your institution` and `Not now`. I read `#421`, `#384`, `#535` and `#217`, the card's design and build tickets. None covers the card after a successful Connect.
- **A2A in-flight:** the 20 newest messages, all read-states. No claim on the setup card.
- **MC:** no prior decision on this case.
- **Own-assignment:** none overlapping.

## The Problem

After a successful Connect, the card stays open above the live cockpit. It shows "Connected. The new PAT is in use, and the Fleet Manager keeps running." and still offers its Create/Connect doors, the plane address field and "Not now".

The operator's words: the card should hide. "Connect" makes no sense once the plane is connected, and "Not now" implies setup is pending although the institution is set up.

## The Architectural Reality

**Design authority:** n/a — defect, with the operator's ruling above as the expected behavior.

**The parity gap:** both doors end in success, but only one retires the card.
- `ConnectContainer` (`apps/agentos/view/setup/ConnectContainer.mjs`) sets the success status and fires `connected` on a reply of `{ok: true}` without `relaunching`.
- `Panel#onConnected` relays the event, and `ViewportController#onSetupConnected()` (`apps/agentos/view/ViewportController.mjs`) handles it. It re-drives the cockpit with `reconnectFleet()` and never touches the card (`reference: 'plane-setup'`).
- The Create door's success does retire the card. `Panel#onFirstPersistence` relays to the owner, which "retires the card from the primary slot".

## The Fix

In `ViewportController#onSetupConnected()`, hide the `plane-setup` card after `reconnectFleet()`, as the Create door's success already does.

Leave these alone:
- The failed-Connect path keeps the card open with its reason text.
- "On request" still brings the card back: the banner's Connect, the shell switcher and Home's doors.

## Acceptance Criteria

- [ ] After a Connect whose reply is `{ok: true}` without `relaunching`, the card is hidden and the cockpit reconnects. A spec mocks `Neo.main.addon.ShellPlane.attachPlane`.
- [ ] After a failed Connect, the card stays open and shows the reason. Same spec.
- [ ] The banner's Connect still reopens the card afterwards.

## Out of Scope

- The banner tone of a refused PAT, yellow versus red (a design note on #424).
- The `relaunching` path, which already ends in a shell restart.

## Related

- #424: row 5, where the walk found it. Not a sub-issue: H3 passed, and this friction does not gate the row.
- `#843`: a PAT update must not restart the FM.
- `#421` and `#384`: the setup card.

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Origin Session ID: b261f993-5994-4a53-bb9c-9fac974a9029
Retrieval Hint: "setup card stays open after successful Connect onSetupConnected"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-11T18:27:14Z @neo-opus-ada added the `bug` label
- 2026-11-11T18:27:15Z @neo-opus-ada added the `good first issue` label
- 2026-11-11T18:27:15Z @neo-opus-ada added the `ai` label
- 2026-11-11T18:39:49Z @neo-opus-ada cross-referenced by #920
- 2026-11-13T22:37:06Z @neo-gpt-emmy added the `help wanted` label
- 2026-11-13T22:37:42Z @neo-gpt-emmy cross-referenced by #936

