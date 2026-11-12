---
id: 932
title: An OS notification tells the operator a merge or a question waits
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-11-12T05:33:58Z'
updatedAt: '2026-11-12T12:56:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/932'
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
closedAt: '2026-11-12T12:56:23Z'
---
# An OS notification tells the operator a merge or a question waits

## Context

Operator, 2026-11-12, correcting my claim that an operator cannot receive wakes: a high-priority wake to the operator can raise a macOS notification. Two kinds of work wait on the operator alone: a pull request that is approved, green and mergeable, and a question a peer addressed to the operator. The operator asked for a ticket; Nightshift (#929, `neomjs/neo-agent-brain#1625`) decides when such a notification should stay quiet.

#551 left push notifications out of scope on 2026-10-04: "the Mailbox is the inbox first, measured, then automated" (D#19394 option E). The inbox shipped (#551, Home's line #557), and this is the automation step. #551's exclusions still bind it: no new daemon, no wake route for the operator, no second credential.

Roadmap disposition: it serves row 4's human merge (a merge-ready pull request reaches the operator who merges it), but no row's installed check needs it. Team tooling the operator requested on 2026-11-12, outside milestone FM v1.

## The Problem

Both kinds of waiting work are visible only inside the cockpit: the fleet head's `N awaiting merge` button (`AwaitingMergeButton`) and the Mailbox's open questions. Three facts keep them there:

- **The cockpit reads only while visible.** `LivenessCadence` launches no read while the cockpit is hidden, so a minimized FM observes nothing new.
- **The installed FM carries no wake for its viewer.** `ViewerWakeController#ensureViewerWakeStream` stamps "wake push not wired" in the packaged shell, whose bridge has no `openWakeStream`.
- **The plane wakes seats, never the operator.** `holderOf` (`neo-agent-brain/ai/services/fleet/openWorkHolder.mjs`) returns `{role: 'operator', kind: 'merge', ids: []}`, and `openWorkWakes` builds wakes for rotation, reviewer and author holders only (`openWorkWakes.mjs`, the `seats` filter).

So a merge-ready pull request waits until the operator happens to look at the cockpit.

## The Architectural Reality

- `holderOf`'s own contract: "`operator` and `rotation` are roles that name nobody; the reader that renders or wakes resolves them". The FM is the operator's reader.
- FM main (`harness/main.mjs`) starts the Fleet child, holds its bearer, and owns the packaged topology's only renderer→Fleet route (`createFleetCapability`, `harness/fleetCapability.mjs`), wire contracts included.
- The verbs main needs exist on the wire (`neo-agent-brain/src/fleet/contract/wire.mjs`), and they are the Home line's own two axes (#557): `fleetOpenWork` (the merges the operator holds) and `fleetOwnQuestions` (the operator's open questions: the viewer's non-terminal A2A Tasks, highest priority first). `fleetRoster`'s DTO carries `operatorPresence` since `neomjs/neo-agent-brain#1632`.
- The renderer's routes are its keeper-view tabs (`ViewportController.config.routes`, pinned by its spec to the rail), and `#/fleet` holds both the merge queue and the Mailbox.
- No module under `harness/` uses Electron's `Notification` today.

## The Fix

1. `harness/operatorNotifications.mjs`, wired in `main.mjs` and listed in `harness/electron-builder.yml` `files:`. On its own cadence, independent of the window's visibility, it reads `fleetOpenWork` and `fleetOwnQuestions` through main's Fleet transport. It keeps the merge-ready rows (repository, number, head) and the open question ids it has seen; the first read after launch baselines them, so a launch raises nothing.
2. A new merge-ready row, or a new question whose priority is `high`, raises one Electron `Notification` (title plus one line) unless an FM window has focus.
3. While `operatorPresence.mode` is `away`, it raises nothing. The first read after the mode returns to `present` raises one summary of what waited ("2 merges and 1 question waited").
4. A click shows the retained cockpit on its Fleet view (`#/fleet`). Opening the exact menu or question would need routes that are not tabs, which the routes-equal-tabs invariant refuses; that refinement waits for an ask.

## Acceptance Criteria

- [ ] A merge-ready row that appears while the operator is present raises one notification; the same row on a later read raises none; the first read after launch raises none.
- [ ] A new high-priority question to the operator raises one notification; an ordinary one raises none.
- [ ] Nothing is raised while away; the return raises one summary, or none when nothing waited.
- [ ] Nothing is raised while an FM window has focus.
- [ ] A click focuses the FM on its Fleet view. `[L3-deferred — operator handoff needed]`: the click path is unit-tested (`createNoticePresenter`); the OS click waits for the installed FM (Post-Merge Validation, owner `neomjs/neo-agent-brain#1508`).
- [ ] Specs cover the transition set: baseline, new, repeat, away, return, focused.

## Post-Merge Validation

- [ ] On the installed FM, a pull request that turns merge-ready raises one macOS notification, and its click opens the FM on the Fleet view.

## Out of Scope

- Waking seats: the plane's open-work wakes stay as they are.
- The viewer wake stream in the packaged shell. This reader needs neither the stream nor a class-3 MC credential in main.
- Badging the tray icon, and notification styling on other platforms.

## Avoided Traps

- **The plane resolving `operator` to the human identity and sending a wake.** No receiver delivers a wake to a person, and main would need the operator's wake subscription and a class-3 MC credential: the wake route and second credential #551 excluded. `holderOf` already names the reader as the place that resolves the role.
- **A notification daemon or an operator channel** (#551's own trap): the reader lives in FM main, beside the Fleet child it already owns.
- **The renderer raising web notifications from the cockpit's reads.** Those reads stop while the window is hidden, which is exactly when a notification matters.
- **`osascript display notification` from the host wake receiver.** It is attributed to Script Editor and has no click target.

## Related

#551 (the operator's inbox; its Out of Scope deferred this), #557 (Home's line), #929 (Nightshift toggle), `neomjs/neo-agent-brain#1625` (operator presence), `neomjs/neo-agent-brain#1632` (the roster's `operatorPresence`), `neomjs/neo-agent-brain#759` (own-work transitions reach the seat), #414 (row 4).

Live latest-open + closed sweep: Institution 60 open (red band; latest 20 read, `#929`..`#842`), closed searches `notification` and `merge ready operator` at 2026-11-12T05:31Z, re-read before filing; the same-surface hit is #551, whose Out of Scope and Avoided Traps are folded in above. No equivalent ticket.
A2A in-flight claim sweep: latest 30 messages at 05:31Z, all read-states; no claim on operator notifications.
MC sweep: "operator OS notification toast for wakes", "holderOf merge role names nobody", "operator away digest at return"; no prior decision.
Own-assignment sweep: 16 open, none on this surface (#929 is the Nightshift toggle this reader consults).

Decision Record impact: none. It follows #551's ruling (inbox first, then automated) and adds no mechanism to the plane.

Owner: Ada (self-assigned 2026-11-12; the operator asked me to file it).

Origin Session ID: 50b913e7-d81f-47b7-b274-84fbf2a6588c
Retrieval Hint: "operator OS notification merge-ready question FM main Electron Notification Nightshift quiet"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-12T05:33:59Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-12T05:34:00Z @neo-opus-ada added the `enhancement` label
- 2026-11-12T05:34:00Z @neo-opus-ada added the `agent-os` label
- 2026-11-12T05:34:01Z @neo-opus-ada added the `ai` label
- 2026-11-12T06:05:01Z @neo-opus-ada cross-referenced by PR #934
- 2026-11-12T06:43:32Z @neo-opus-ada referenced in commit `cc6250e` - "fix(harness): notices need real evidence and outlive the banner (#932)

An ok answer whose result is unavailable (open work before the
producer's first poll, a failed questions read) is no evidence: it
neither baselines nor drops what was seen, and the health line names
it. Merge rows are keyed by repository, number and head and every seen
key is kept (bounded), so a mergeable flicker stays quiet while a new
head raises again. Shown notifications stay referenced until clicked or
closed; the presenter owns the click path to the Fleet view."
- 2026-11-12T12:56:23Z @tobiu referenced in commit `a391cfa` - "feat(harness): an OS notification tells the operator what waits (#932) (#934)

* feat(harness): an OS notification tells the operator what waits (#932)

Electron main reads the operator's waiting work on its own 60 s cadence:
the open-work producer's awaitingMerge, the operator's open questions,
and the roster's operatorPresence, through a new main-internal
fleetCapability.read (same transport, bearer and secret census as a
renderer call; never a credential verb). The cockpit's own reads stop
while its window hides, and the packaged shell carries no viewer wake.

The planner is pure. The first answer of each kind baselines, so a
launch raises nothing. A new merge or a new high-priority question
raises one Notification. While the operator is away, it counts instead,
and the return raises one summary. A focused FM window raises nothing,
and a failed read changes nothing. A click shows the cockpit on
#/fleet. The module joins electron-builder's files list and the
packaged-main closure.

* fix(harness): notices need real evidence and outlive the banner (#932)

An ok answer whose result is unavailable (open work before the
producer's first poll, a failed questions read) is no evidence: it
neither baselines nor drops what was seen, and the health line names
it. Merge rows are keyed by repository, number and head and every seen
key is kept (bounded), so a mergeable flicker stays quiet while a new
head raises again. Shown notifications stay referenced until clicked or
closed; the presenter owns the click path to the Fleet view."
- 2026-11-12T12:56:24Z @tobiu closed this issue

