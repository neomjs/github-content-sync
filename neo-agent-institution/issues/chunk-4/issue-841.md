---
id: 841
title: 'Cold relaunch: the roster read times out while the fleet is ready'
state: OPEN
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-30T21:08:03Z'
updatedAt: '2026-10-30T21:08:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/841'
author: neo-opus-vega
commentsCount: 0
parentIssue: 732
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
# Cold relaunch: the roster read times out while the fleet is ready

## Context

Row 3's sitting (#485, 2026-10-30) began with the cold relaunch of the installed Neo Harness (Institution `20d89be` / Brain `1bf8bd91` / engine `477b65bd9e`). The operator recorded it; the harness log and the app console give the rest. All times UTC.

| Time | Side | Event |
|---|---|---|
| 19:42:53.7 | harness `main.log` | SIGINT; continuity handoff over 8 running seats, sealed after 3,337 ms |
| 19:43:22.5 | recording | new window up; "fleet connecting · Roster read in progress — no fleet data yet", 0 agents |
| 19:43:24.2 | harness `main.log` | app↔fleet transport listening on 8083; Neural Link Bridge adopted |
| 19:43:30.4 | harness `main.log` | boot phase runtime verify: 6,229 ms |
| 19:43:32.1 | harness `main.log` | continuity recovery: 13 retained generations, 1,656 ms |
| 19:44:17 | recording (1:30) | **"fleet timed out · Roster read timed out — no fleet data yet"**, 0 agents, "plane admission unknown" |
| 19:44:28 | recording (1:41) | roster lands: 13 agents, "8 started · 3 excluded", "feed partial" |

The roster appeared 65 s after the window; the fleet was ready 9 s after it. The renderer's console shows brain-health polls at +0.3 s and +1.3 s and the next one at +120 s. The operator's words: "it took VERY long to reconnect and show the roster." Second data point from the same sitting: a View → Reload at 20:45:43Z re-read the Observatory scene in 19 s, so the cost is specific to the roster read after a relaunch.

## The Problem

The cockpit's first roster read after a continuity recovery runs against a fleet that is still binding (runtime verify 6.2 s, recovery 1.7 s), gives up on a fixed budget, tells the operator the fleet "timed out", and retries only about a minute later. The fleet is up the whole time; the message is wrong and the wait is the retry cadence, not the fleet.

## The Architectural Reality

- The harness Brain logs the fleet's boot phases (runtime bind 45 ms, viewer claim, plane init 332 ms, transport 34 ms, runtime verify 6,229 ms, recovery 1,656 ms) from `harness/brain.mjs`; the transport answers before the phases finish.
- The Fleet cockpit reads the roster over that transport; its read budget, its "timed out" line and its retry cadence live in the cockpit's reading surface (`apps/agentos/view/fleet/cockpit/ReadingSurfacesController.mjs` and the roster store), to be read before the fix names a number.
- Nothing in the renderer consumes the boot phases the fleet already reports, so the cockpit cannot say "binding, recovering 13 seats" and waits blind.

## The Fix

The renderer waits for the fleet's ready signal, or retries with a short backoff while the transport reports a boot phase, instead of one fixed timeout and a long retry; while binding, the state line names the phase ("fleet binding · recovering 13 seats") and never "timed out" for a fleet that answers.

## Acceptance Criteria

- [ ] A unit or NL arm: a fleet that becomes ready 9 s after the window yields a roster within 15 s and never shows "timed out"; red on `dev` first.
- [ ] Installed receipt on the next cold relaunch: window-to-roster time recorded on this ticket, 15 s or less.
- [ ] While the fleet binds, the state line names the boot phase it is in.

## Out of Scope

The fleet's own 6.2 s runtime verify (a Brain measurement, its own ticket if it grows). The stale `backup-success-overdue` state Ada noted the same day. The Observatory's own reads (#485's checks).

## Related

#732 (an FM update never requires closing a harness; this relaunch is the same journey after the update). #534 (row 1: a cold first run reaches done). neomjs/neo-agent-brain#964 (a start-fleet proof race, a sibling by shape). #424 (an ordinary failure returns to live by the product's own guidance). Receipt: #485.

Live latest-open sweep: latest 20 open Institution issues read at 2026-10-30T21:06:46Z; no equivalent. A2A sweep: last 30 `lane-claim` messages, none on the roster read. MC sweep: "roster read timed out fleet connecting no fleet data yet cold launch continuity recovery renderer retry", 5 results, no prior decision (Brain #964 is a start-fleet race, cited above). Own-assignment sweep: 4 open (#732, #651, #649, #485); #732 is the umbrella journey, not this read.

Origin Session ID: 40918982-6f4d-45f6-8b01-c5f951c22edf
Retrieval Hint: "cold relaunch roster read timed out 65 seconds fleet ready in 9 continuity recovery"

## Timeline

- 2026-10-30T21:08:04Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-30T21:08:05Z @neo-opus-vega added the `bug` label
- 2026-10-30T21:08:06Z @neo-opus-vega added the `agent-os` label
- 2026-10-30T21:08:06Z @neo-opus-vega added the `ai` label
- 2026-10-30T21:09:14Z @neo-opus-vega added parent issue #732
- 2026-10-30T21:10:05Z @neo-opus-vega cross-referenced by #485
- 2026-10-30T21:10:48Z @neo-opus-vega cross-referenced by #312
- 2026-10-30T22:39:13Z @neo-opus-ada cross-referenced by #516
- 2026-10-30T22:39:28Z @neo-opus-ada cross-referenced by #424
- 2026-10-30T22:40:40Z @neo-opus-ada cross-referenced by #843
- 2026-10-30T22:44:48Z @neo-opus-ada cross-referenced by #844

