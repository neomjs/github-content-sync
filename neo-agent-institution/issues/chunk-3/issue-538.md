---
id: 538
title: 'The Institution pins Brain 786d9c4a: the producers the next walks read'
state: CLOSED
labels:
  - enhancement
  - ai
  - dependencies
assignees:
  - neo-opus-grace
createdAt: '2026-10-04T12:13:31Z'
updatedAt: '2026-10-04T12:39:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/538'
author: neo-opus-grace
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
closedAt: '2026-10-04T12:39:15Z'
milestone: FM v1
---
# The Institution pins Brain 786d9c4a: the producers the next walks read

## Context

The Institution pins Brain `5d466610`, set by #503 via PR #515. Brain `dev` is at `786d9c4a`, eight merges later. The next #12 candidate needs that Brain: every FM v1 walk planned on it reads at least one of these producers (peer read on #12, comment 5979698752).

| Brain merge | What reaches the Institution | Read by |
|---|---|---|
| neomjs/neo-agent-brain#824 | Lane claims reach the roster card and stay until replaced | row 4, step 1 (#490) |
| neomjs/neo-agent-brain#835 | Each seat's open work is read with its own PAT, so PR events arrive without `GH_TOKEN` | row 4, steps 2–4; row 2's shared gap 3 |
| neomjs/neo-agent-brain#832 | The computed route carries the synthesizer's run id | row 3, #510 AC-3 (#485) |
| neomjs/neo-agent-brain#828 | A desktop seat's status names where its session opened | #522 |
| neomjs/neo-agent-brain#827 | The Fleet serves existing agents' memory candidates (`fleetMemoryCandidates`) | #521 |
| neomjs/neo-agent-brain#817 | The presence hook writer reads its deadlines from the turnPresence leaves | seats re-project their hook (rollout note on #817) |
| neomjs/neo-agent-brain#831 | The Brain consumes Skills 0.1.29 | — |
| neomjs/neo-agent-brain#821 | Heartbeat and idle-nudge texts name the accepted plan's next step | wake texts |

## The Problem

A merged Brain producer reaches no installed walk until the Institution pins a Brain that contains it. Row 4's walk failed on steps 1–4 on 10-03, and the producers that fix those steps (#824, #835) are now merged. Today's pin still predates every one of them: each is checked with `git merge-base --is-ancestor <merge> 5d466610`, and each check is false.

## The Architectural Reality

Delta census: `git diff --stat 5d466610 786d9c4a -- src package.json`, read by rows.
- `src/fleet/contract/wire.mjs`: `FLEET_WIRE_METHODS` gains `fleetMemoryCandidates` (#827). The Institution consumes that list in `apps/agentos/fleet/installFleetBridge.mjs:247`, `harness/brain.mjs:125` and `harness/main.mjs:461`. The new verb becomes callable, and nothing calls it until #521.
- `package.json`: the Brain's own `neo-agent-skills` range moves from `^0.1.23` to `^0.1.29`. The Institution's own skills dependency is untouched.

## The Fix

- **Point the pin** at `786d9c4a` (full sha) in `package.json`, `package-lock.json` and `.github/workflows/ci.yml` (the Brain checkout `ref`).
- **Reinstall** the dependency (`rm -rf node_modules/neo-agent-brain && npm install`) and check one marker from the delta: the installed `src/fleet/contract/wire.mjs` lists `fleetMemoryCandidates`.
- **Run CI's jobs:** unit, components, E2E, and the cross-repository job (Brain at the pin, `NEO_AGENTOS_RUNTIME_ROOT`).

## Acceptance Criteria

- [ ] AC-1: all three places name `786d9c4a`, and the installed module carries the marker.
- [ ] AC-2: the Institution's unit, components, E2E and cross-repository CI jobs pass on the new pin.

## Post-Merge Validation

- [ ] The next #12 candidate is built on this pin. Its manifest check (Grace, verifying Emmy's freeze) reads each producer above as an ancestor of the frozen Brain pin.

## Out of Scope

- The Institution consumers of the new producers (#521, #522, #524, row 2's gap 3 words): each is its own leaf.
- Freezing, building or installing the candidate (#12, Emmy).

## Related

#12 (the candidate) · #414 (row 4) · #477 (row 2) · #312 (row 3) · precedent #469.

Decision Record impact: `none`.

Sweeps: live latest-20 open Institution issues at 12:14Z: no pin ticket. A2A, all read states, last 60 min: no pin claim except mine to Emmy at 12:12Z. Memory Core: no prior decision on this pin. Own assignments: none on this surface.

Origin Session ID: 8a9af488-83ca-46fa-8ad4-41448d022b45
Retrieval Hint: "Institution pins Brain 786d9c4a candidate producers row 4 walk"

## Timeline

- 2026-10-04T12:13:31Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-04T12:13:32Z @neo-opus-grace added the `enhancement` label
- 2026-10-04T12:13:33Z @neo-opus-grace added the `ai` label
- 2026-10-04T12:13:33Z @neo-opus-grace added the `dependencies` label
- 2026-10-04T12:13:46Z @neo-opus-grace added this to the **FM v1** milestone
- 2026-10-04T12:15:54Z @neo-opus-grace cross-referenced by PR #539
- 2026-10-04T12:19:28Z @neo-gpt-emmy cross-referenced by #12
- 2026-10-04T12:39:14Z @tobiu referenced in commit `22724d4` - "feat(deps): pin Brain 786d9c4a — the producers the next walks read (#538) (#539)

Lane claims on the card (#824), per-seat PR reads (#835), the Golden Path run id (#832), the desktop seat's session folder (#828) and existing agents' memory candidates (#827) reach the next candidate. The pin moves in package.json, the lock and the CI Brain checkout."
- 2026-10-04T12:39:15Z @tobiu closed this issue

