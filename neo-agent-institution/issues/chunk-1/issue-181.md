---
id: 181
title: An instance switch keeps the previous instance's roster and activity
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-09-23T08:50:12Z'
updatedAt: '2026-09-23T10:41:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/181'
author: neo-fable-clio
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
closedAt: '2026-09-23T10:39:10Z'
---
# An instance switch keeps the previous instance's roster and activity

## Context

On 2026-09-19 a real switch to an unreachable instance (the `#173` post-merge witness, [receipt](https://github.com/neomjs/neo-agent-institution/pull/173#issuecomment-5742436024)) kept the previous instance's nine resident cards and its activity rows on screen under the new instance's name, labelled `stale`. It was captured as a defect-note that day and left unpromoted because it was unexamined whether a *reachable* second instance replaces the rows.

Promoted now by triage: D#18965's STEP_BACK ([sweep 4](https://github.com/neomjs/neo/discussions/18965#discussioncomment-18516216), @neo-gpt-emmy) makes target-bound evidence a graduation blocker for the first-run journey, and this is the cockpit's live instance of that invariant. Today's source read answers the open question: a reachable instance replaces the roster, but not the activity feed.

Sweeps at 2026-09-23T08:49Z: latest open Institution issues — nearest is #15 (banner vocabulary), not equivalent; A2A — no claim on this scope; Memory Core — only the 09-19 capture.

## The Problem

`ViewportController.switchToProfile()` promises "the cockpit's own full re-drive behind its generation fences (no cross-instance bleed by construction)". The fences drop a *late* answer from the previous bridge. They do nothing about rows already in the stores:

1. **Roster, unreachable or malformed B** — a throw goes to `degradeWiredSurface('grid')` → `stale`; a malformed answer returns early. Either way the store still holds A's residents, and `rosterWired` / `lastLiveRows` still carry A's snapshot. `stale` means "last-known rows of this target"; these rows belong to another target. *Runtime-witnessed.*
2. **Activity, reachable B** — `ingestSnapshot(events, {replace: !me.activityWired})`: `activityWired` is still `true` from A, and the store never treats omission as deletion. B's page joins A's retained events, and the feed interleaves two institutions under B's name until the retention bound evicts A's tail. *Code-derived, not yet runtime-witnessed.*
3. **Activity, unreachable B** — the same `stale` retention as the roster. *Runtime-witnessed (same run as 1).*

## The Architectural Reality

- `apps/agentos/view/fleet/cockpit/LivenessController.mjs` — `lastLiveRows` :97, `rosterWired` :126, `activityWired` :144; `loadActivity()` admits with `replace: !me.activityWired` at :253; `loadRoster()` returns early on a malformed answer at :326 and degrades at :388; `degradeWiredSurface()` :475 picks `stale` whenever the surface was ever `live`; the seed-load guard at :657 re-applies `lastLiveRows`; `reconnectFleet()` :802 is the switch re-drive and resets nothing.
- `apps/agentos/store/FleetActivityEvents.mjs` `ingestSnapshot()` :85 — reconciles "without interpreting omission as deletion"; `replace` exists for the sample on first live admission only.
- The existing provenance primitive: `apps/agentos/util/DeploymentStateRead.mjs` stamps its picture with the answering `bridge.profileId` — "a picture can only land stamped with the profile that answered it". Every bridge carries `profileId` (`fleet/installFleetBridge.mjs` :72, `fleet/fleetSessionCustody.mjs` :98 / :120).

## The Fix

Bind retained truth to the target that produced it, the way `DeploymentStateRead` already does:

- Roster and activity each record the `profileId` of the bridge whose answer they admitted.
- A read through a bridge with a different `profileId` first retires the previous target's truth: its rows leave the store, `rosterWired` / `activityWired` reset, `lastLiveRows` drops. B's first live answer is then a first admission, and B's failure shows B's own state with no rows. The state word for "no answer from this target yet" is a design call made in the PR; `stale` is ruled out because it claims last-known rows of this target.
- A same-target failure keeps its last-known rows as `stale`, unchanged.
- The `switchToProfile()` JSDoc then holds, or is corrected.

Unverified neighbours, checked in the same pass: the memories, catch-up and wake-routes panes (re-driven via `onRefreshClick`), tasks, operator identity. A neighbour joins this PR only if its fix is the same stamp; anything else becomes a defect-note.

## Acceptance Criteria

- [ ] Unit, red-first: A admits a roster → the bridge's `profileId` changes → B's read throws → no A resident remains in the store, and the grid does not read `stale`.
- [ ] Unit, red-first: the same for a malformed B answer.
- [ ] Unit, red-first: A admits activity → switch → B answers `wired` → the store holds exactly B's events.
- [ ] Unit control: a same-target transient failure still keeps last-known rows as `stale`.
- [ ] Unit control: a same-target second live page still reconciles; omission and retention semantics unchanged.
- [ ] Live witness (L3), state layer: in the running cockpit, a switch to a dead endpoint shows the seed roster under its own word (`FLEET · 11 AGENTS static roster`, the seed-only ids present) and an empty activity feed (`0 retained · sample`), never A's rows as `stale`.
- [x] Live witness (L3), DOM layer: no previous resident card node remains after the switch — delivered by #182 (PR #185): on the combined head `8b06921` (#184@d21cf67 + #185@06f62a4) the switch shows 11 seed card nodes, one per key, zero previous-instance cards. This line is #182's deliverable; this ticket only points at its receipt.
- [ ] Switching back to A reloads A's truth — blocked by a pre-existing manager defect (`connectinstance` / `saveinstance` reach the ViewportController with `source` as an id string; defect-note filed 2026-09-23); the unit arm "a reachable new profile is admitted as a FIRST snapshot" carries the state-layer proof.
- [ ] No visual golden moves.

## Out of Scope

- The first-run journey's recipe and evidence contract (D#18965).
- The banner vocabulary (#15).
- Tear-out window titles — already re-titled per switch by `pushVesselTitles()`.

## Decision Record impact

`none` — aligns the roster and activity reads with the provenance rule `DeploymentStateRead` already follows.

## Related

D#18965 · #173 · #15 · #10

Origin Session ID: f34cbeb6-fd44-4060-b31f-e05332e62aee
Retrieval Hint: "instance switch keeps previous instance roster activity stale target-bound evidence"




## Timeline

- 2026-09-23T08:50:13Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-23T08:50:14Z @neo-fable-clio added the `bug` label
- 2026-09-23T08:50:14Z @neo-fable-clio added the `agent-os` label
- 2026-09-23T08:50:14Z @neo-fable-clio added the `ai` label
- 2026-09-23T09:22:35Z @neo-fable-clio cross-referenced by #182
- 2026-09-23T09:22:46Z @neo-fable-clio cross-referenced by #183
- 2026-09-23T09:23:46Z @neo-fable-clio cross-referenced by #42
- 2026-09-23T09:23:56Z @neo-fable-clio cross-referenced by PR #184
- 2026-09-23T09:27:13Z @neo-fable-clio referenced in commit `2d5524b` - "fix(cockpit): keep the ticket archaeology out of the TargetBinding source comment (#181)"
- 2026-09-23T09:27:30Z @neo-fable-clio referenced in commit `d21cf67` - "test(visual): refresh the baseline input stamp for the TargetBinding comment (#181)"
- 2026-09-23T10:01:16Z @neo-fable-clio cross-referenced by PR #185
- 2026-09-23T10:11:48Z @neo-fable-clio cross-referenced by PR #186
- 2026-09-23T10:39:10Z @tobiu closed this issue
- 2026-09-23T10:47:33Z @neo-fable-clio referenced in commit `229abaa` - "fix(cockpit): retained roster and activity truth belongs to the profile that answered (#181)"
- 2026-09-23T10:47:33Z @neo-fable-clio referenced in commit `a512543` - "fix(cockpit): keep the ticket archaeology out of the TargetBinding source comment (#181)"
- 2026-09-23T10:47:33Z @neo-fable-clio referenced in commit `d6252fe` - "test(visual): refresh the baseline input stamp for the TargetBinding comment (#181)"
- 2026-09-23T10:47:33Z @neo-fable-clio referenced in commit `fe2912f` - "fix(viewport): the product injector stamps the endpoint's canonical profile identity onto the bridge it installs (#181)"
- 2026-09-23T11:05:53Z @neo-fable-clio referenced in commit `683ff98` - "fix(cockpit): retained roster and activity truth belongs to the profile that answered (#181)"
- 2026-09-23T11:05:53Z @neo-fable-clio referenced in commit `ee23954` - "fix(cockpit): keep the ticket archaeology out of the TargetBinding source comment (#181)"
- 2026-09-23T11:05:53Z @neo-fable-clio referenced in commit `77b3a10` - "test(visual): refresh the baseline input stamp for the TargetBinding comment (#181)"
- 2026-09-23T11:05:53Z @neo-fable-clio referenced in commit `392a8d9` - "fix(viewport): the product injector stamps the endpoint's canonical profile identity onto the bridge it installs (#181)"
- 2026-09-23T11:48:28Z @tobiu referenced in commit `7531a1c` - "Merge pull request #184 from neomjs/agent/181-instance-switch-target-binding

fix(cockpit): retained roster and activity truth belongs to the profile that answered (#181)"

