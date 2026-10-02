---
id: 460
title: 'The Institution pins Brain 8a078f0: open work, seat plane, bounded reasons'
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-02T18:03:36Z'
updatedAt: '2026-10-02T18:59:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/460'
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
closedAt: '2026-10-02T18:59:57Z'
---
# The Institution pins Brain 8a078f0: open work, seat plane, bounded reasons

## Context

The Institution pins Brain `447d96e` (#451 via PR #452). Brain `dev` is at `8a078f0`, seven merges later. Emmy handed the pin to me after her engine pin (#458) merged.

| Brain merge | What reaches the Institution |
|---|---|
| neomjs/neo-agent-brain#764 | the open-work producer and the `fleetOpenWork` read verb; Institution #449's prerequisite |
| neomjs/neo-agent-brain#771 | seat hooks reach the plane as the seat: `AiConfig.seat` plus the `NEO_SEAT_PLANE_BASE` reserved launch slot. neomjs/neo-agent-brain#766's PMV-1 needs a Fleet Manager on this pin. |
| neomjs/neo-agent-brain#774 | lifecycle `failureReason` carries a refusal's words or a code, never a caught message |
| neomjs/neo-agent-brain#769 | the PR lane carries the producer's transitions |
| neomjs/neo-agent-brain#770 / #775 | the OpenAI-compatible client sends request fields only; Gemini Flash defaults move to `gemini-3.8-flash` |
| neomjs/neo-agent-brain#754 | an explicit public-summary recency option (`memorySharing`) |

## Delta census (`git diff --stat 447d96e 8a078f0 -- src package.json`, read by rows)

- One row: `src/fleet/contract/wire.mjs`, where `FLEET_WIRE_METHODS` gains `fleetOpenWork`. No `package.json` change.
- The Institution imports `FLEET_WIRE_METHODS` from `neo-agent-brain/fleet-contract`, so `installFleetBridge` exposes the verb automatically. `installFleetBridge.spec`'s key-parity test follows the import.
- No Institution source or test copies a method list or #774's wording.
- `apps/agentos/design/first-run-setup-card.html` (a static design page) still names `gemini-3.5-flash`. That is out of scope here and noted to its owner.

## The Fix

- Point the pin at `8a078f0` in `package.json`, `package-lock.json` and `.github/workflows/ci.yml` (the Brain checkout `ref`).
- Reinstall the dependency and check a marker from the delta.
- Run CI's jobs: the unit, components and E2E jobs, plus the cross-repository job (Brain at the pin, one shared physical engine, `NEO_AGENTOS_RUNTIME_ROOT`).

## Acceptance Criteria

- [ ] AC-1: all three places name `8a078f0`, and the installed dependency carries `fleetOpenWork` in its wire contract.
- [ ] AC-2: both CI jobs are green: the unit, components and E2E job, and the Brain-bound unit contract with the E2E list.

## Out of Scope

- Consuming `fleetOpenWork` (#449).
- The static design page's preset text.
- #766's installed check (PMV-1, owned by neomjs/neo-agent-brain#30).

## Sweeps

- Live latest-open: the latest 20 open Institution issues at 2026-10-02T18:03Z, plus a "Brain pin" search; no equivalent.
- A2A: no pin claim besides Emmy's handoff (17:58Z).

Origin Session ID: 6f7d14a3-e126-4b47-888f-fc28c748ae83

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-10-02T18:03:37Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-02T18:03:38Z @neo-opus-ada added the `enhancement` label
- 2026-10-02T18:03:38Z @neo-opus-ada added the `ai` label
- 2026-10-02T18:12:11Z @neo-opus-ada cross-referenced by PR #462
- 2026-10-02T18:25:39Z @neo-gpt-emmy cross-referenced by #42
- 2026-10-02T18:59:57Z @tobiu referenced in commit `1ac1273` - "feat(deps): pin Brain 8a078f0 — open work, seat plane, bounded reasons (#460) (#462)"
- 2026-10-02T18:59:57Z @tobiu closed this issue
- 2026-10-02T20:32:01Z @neo-opus-ada cross-referenced by #469

