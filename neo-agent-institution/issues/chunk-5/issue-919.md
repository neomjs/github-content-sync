---
id: 919
title: The activity feed reaches live minutes after the roster
state: OPEN
labels:
  - bug
  - ai
  - performance
assignees: []
createdAt: '2026-11-11T18:39:39Z'
updatedAt: '2026-11-12T02:14:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/919'
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
# The activity feed reaches live minutes after the roster

## Context

The operator and I saw this on 2026-11-11, on FM Institution `6f00d3da` / Brain `7219f4e1` / Engine `c7e19744`, during row 5's walk. The operator's words: "roster showed up reasonably fast, but it took a lot longer until the activity feed content got shown." The defect-note fingerprint is `b4be35b967093242`.

The operator saw it again on 2026-11-12 after the install of Institution `499339f` / Brain `b991bd5`: "it took still VERY long after the roster showed until the activity feed got content. why? this is friction." That install's Fleet transport came up at 02:01:17.9Z and recovery finished at 02:01:37.8Z. Defect-note fingerprint `8673eded8259ffd8`.

**Sweeps, 18:38:51Z:**
- **Live latest-open + closed:** 54 open, red band. Searched `activity read timed out`, `feed partial` and `activity feed lag`. Only the row tickets (#424, #477, #516) mention the state, and none owns the lag.
- **A2A:** the 12 newest messages, all states. No claim.
- **MC:** no prior decision.

## The Problem

Three starts on the same machine (load average 15–16). Window-only frames every 2 s and Neural Link reads of `SpineBannerComponent`.

| Start | Fleet booted | Roster live | Activity (`feed partial` → live) | Activity lag behind roster |
|---|---|---|---|---|
| Install relaunch (H0) | 17:38:00Z (runtime verify ended 17:38:25Z) | about 17:38:39Z | `feed partial` from 17:40:21Z, live 17:41:20Z | about 2.5 min |
| Plane-down relaunch (H1) | 17:56:20Z | about 17:56:4xZ | live about 17:57:27Z | about 40 s |
| Connect after a revoked PAT (H3) | 18:17:00Z | at once | `feed partial` ("PR lanes partial · mailbox available · pr-lane: open-work producer coverage partial") until 18:19:57Z | about 3 min |

The first activity reads time out: "Activity read timed out — roster is live · no activity data yet · fleet read exceeded 10000ms".

**The plane is not the bottleneck.** Its `get_pr_lane_activity` (limit 50, prEvents) answered in under 1 s at 17:58:47Z. The time goes into the FM's first activity reads after boot.

## The Architectural Reality

- **Design authority:** n/a — defect (the observed lag above). Expected: the activity feed is live soon after the roster. The cadence's own record: "the roster and the activity stream within a minute" (`AgentOS.util.LivenessCadence`).
- **The activity stream** is `apps/agentos/view/fleet/activity/Container.mjs` (`ActivityStream`).
- **Its sources** come from the host fleet (port 8083) and the plane: the mailbox, the PR lanes, and the open-work producer's coverage (`pr-lane: open-work producer coverage partial`).
- **The cadence half (found 2026-11-12).** The cockpit fires every liveness read once at mount; after that, each read comes due only after its own interval (activity 60 s, checked on a 15 s pass).
  - While the Fleet's start gate boots, only the roster comes due on every pass (`LivenessCadence.intervalsFor`).
  - `LivenessCadence.expedite` is called only for the roster (`FleetAdmission.mjs`).
  - So an activity read that times out during boot ("fleet read exceeded 10000ms") is not retried until a full interval later, and a failure then costs another.
- **Unknown:** which source holds the feed at `partial` once the reads do answer. It could be cold reads in the fleet child right after runtime verify, or the producer's first round.

## The Fix

1. When the start gate leaves `booting`, bring every liveness read forward to the next pass, not only the roster, so a read that timed out during boot retries within one pass of the Fleet finishing.
2. Measure the `partial` phase. Add per-source timing to the activity read on a cold start: when each source first answers, and when the feed leaves `partial`. Name the slow source.
3. Fix that source's first read so the feed is live within one read of the roster. Do not widen the 10 s timeout.

## Acceptance Criteria

- [ ] A `LivenessCadence` spec: when the start gate goes from `booting` to open, every read comes due within one pass. Red on `dev`.
- [ ] A receipt names the source holding the feed at `partial` after a cold start, with per-source timings.
- [ ] After the fix, on the installed FM, the activity feed is live within one read cycle of the roster on a cold start. One receipt.

## Out of Scope

- The plane's read cost, already measured under 1 s.
- The activity feed's keyboard regression, neomjs/neo#20783.

## Related

- #424, #516 and #477.
- Institution #913: open-work wakes.

unowned-rationale: The measurement step is ready for a peer; maintainers provide review.

Origin Session ID: b261f993-5994-4a53-bb9c-9fac974a9029
Retrieval Hint: "activity feed partial minutes after roster cold start"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-11T18:39:40Z @neo-opus-ada added the `bug` label
- 2026-11-11T18:39:40Z @neo-opus-ada added the `ai` label
- 2026-11-11T18:39:41Z @neo-opus-ada added the `performance` label
- 2026-11-12T00:39:20Z @neo-opus-grace cross-referenced by #925
- 2026-11-12T02:15:43Z @neo-opus-ada cross-referenced by #927
- 2026-11-12T02:15:56Z @neo-opus-ada cross-referenced by #928

