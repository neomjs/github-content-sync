---
id: 518
title: 'fleetServer.spec''s boot-gate arm is red on dev, unseen by CI'
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-grace
createdAt: '2026-09-25T21:30:04Z'
updatedAt: '2026-09-25T21:42:05Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/518'
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
closedAt: '2026-09-25T21:42:05Z'
---
# fleetServer.spec's boot-gate arm is red on dev, unseen by CI

## Context

@neo-fable filed this as a defect-note at 15:58Z (`MESSAGE:220cdda6`). I reproduced it on `dev` 7b04b98 at 21:28Z, the independent second sighting that promotes the note. `test/playwright/unit/ai/services/fleet/fleetServer.spec.mjs` carries two drifted arms. Only the first shows, because the file runs serially (`test.describe.configure({mode: 'serial'})`, `:126`) and nothing after a red runs.

1. **"the boot gate requires the mounted root and rejects absent, relative, or split placement"** fails with:
   ```
   planeConfig.collectPlaneMembers: claimed member "remRunStateDir" is not resolvable in the resolved tree.
   ```
2. **"Fleet S1 wire policy › serves ready verbs…"** fails once (1) passes: its exhaustive `expectedSlices` lacks `adoptAgent` and `releaseAgent`.

## The Problem

- **Arm 1:** `ai/configBase.mjs:2725`'s `PLANE_MEMBER_PATHS` names 13 plane members, among them `remRunStateDir`, the leaf at `:273` that resolves to `rem-runs` under the plane data root. The arm's hand-built `aiConfig` fixture (`:818-843`) places the other 12.
- **Arm 2:** #377 declared `adoptAgent` and `releaseAgent` as `awaiting-s4` lifecycle-write verbs in `ai/services/fleet/fleetServerPolicy.mjs` (`:24-25`, `:72-73`). The arm's mirror of every non-ready verb was never updated.

The spec is not on `.github/workflows/brain-unit.yml`'s run list. `--list` collects it and only the listed specs execute, so CI saw neither drift, and #377's own CI could not have either.

## The Architectural Reality

- `assertFleetPlaneReady({aiConfig, rootDir})` takes an injected config object, and arm 1 hands it a literal. Nothing touches the `AiConfig` singleton (ADR 0019 §3 B4 is not in play).
- Arm 2 closes with `Object.keys(expectedSlices)` against `FLEET_WIRE_METHODS` minus the ready set. That is a completeness mirror, so a new verb must appear in it.

## The Fix

1. Add `remRunStateDir: member('rem-runs')` to arm 1's fixture, beside the other top-level members.
2. Add `adoptAgent` and `releaseAgent` as `'awaiting-s4'` to arm 2's `expectedSlices`. The arm then also proves both refuse at the authority boundary without a subject, as lifecycle-write verbs.
3. Put the spec on `brain-unit.yml`'s run list, so the next plane member or wire verb breaks these arms in the PR that adds it.

## Acceptance Criteria

- [ ] Both arms pass (each red on `dev`).
- [ ] `fleetServer.spec.mjs` runs in CI's Brain unit job, all 23 tests green.

## Out of Scope

- **Deriving either fixture from its source list.** Both arms state their expectations by hand on purpose, so that they test the code against explicit placements and slices.

## Related

#500 · #502 (the `rem-runs` shared channel) · #377 (the two verbs) · #324 (the same fixture-drift class in `FleetServerComposition`)

Decision Record impact: none.
Live latest-open sweep: checked latest 20 open issues at 2026-09-25T21:29:37Z; no equivalent found.
A2A in-flight sweep: no claim on this scope since the 15:58Z note.
MC sweep: "fleetServer.spec boot gate red on dev, remRunStateDir not resolvable…", 5 results. @neo-fable's 17:06Z turn records it as a pre-existing red; no fix in flight.
Own-assignment sweep: none overlapping.

Revised 21:33Z: arm 2 surfaced once arm 1 passed.

Origin Session ID: d2d30528-b6fe-423b-86ce-ab945396a201


## Timeline

- 2026-09-25T21:30:04Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-25T21:30:06Z @neo-opus-grace added the `bug` label
- 2026-09-25T21:30:06Z @neo-opus-grace added the `ai` label
- 2026-09-25T21:30:06Z @neo-opus-grace added the `testing` label
- 2026-09-25T21:33:17Z @neo-opus-grace cross-referenced by PR #519
- 2026-09-25T21:42:05Z @tobiu referenced in commit `60807a5` - "Merge pull request #519 from neomjs/grace/518-fleet-boot-gate-fixture

test(fleet): fleetServer.spec's two drifted arms pass again and the spec joins CI's run list (#518)"
- 2026-09-25T21:42:05Z @tobiu closed this issue

