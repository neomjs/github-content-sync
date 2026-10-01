---
id: 409
title: 'Brain pin 8 and engine pin: the installed FM carries the Fleet''s wake-route arming'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-01T19:23:39Z'
updatedAt: '2026-10-01T20:04:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/409'
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
closedAt: '2026-10-01T20:04:30Z'
---
# Brain pin 8 and engine pin: the installed FM carries the Fleet's wake-route arming

## Context

Pin 7 (#402 / #403) merged at `f21a2d2` with Brain `dev@5041af0`. Two minutes earlier, neomjs/neo-agent-brain#705 merged: the Fleet arms a launched Codex or Claude Desktop seat's wake route as the seat. Its other half, #405 (merged, `6246a89`), hands the installed FM's wake receiver to the Fleet. The two halves meet only once the pin carries #705. #705's installed residual (its AC-9) is owned by neomjs/neo-agent-brain#571.

*Revised 2026-10-01 ~19:28Z:* the engine pin is folded in, on Mnemo's Tier 2.5 ask, so one package carries both pins and Emmy restarts once.

## The Problem

Brain `dev@92122a0` carries four merges the pin does not:
- neomjs/neo-agent-brain#705: wake-route arming for Fleet-launched GUI seats.
- neomjs/neo-agent-brain#716: `NEO_GEMINI_MODEL` / `NEO_GEMINI_EMBEDDING_MODEL` bind the Gemini model leaves.
- neomjs/neo-agent-brain#718 and neomjs/neo-agent-brain#720: two Brain unit-arm flake fixes.

The engine pin (`neomjs/neo@e7d550e5dc`) also trails engine `dev@08ff2a55e6` by two `src` changes:
- neomjs/neo#19351: the dock Workspace composes its default cross-window Participation for a published sort group. Its AC-5 asks that the Institution's next engine pin keep `FleetCockpitTabDragIndicatorsNL` green.
- neomjs/neo#19340: a reveal overlay's inside mousedown leaves a focusable target's focus alone.

## The Architectural Reality

The Brain pin moves in three places together, as pins 5–7 did: `package.json`, `package-lock.json`, and CI's Brain checkout in `.github/workflows/ci.yml`. Between `5041af0` and `92122a0` nothing under the Brain's `src/` (the Body-safe contract) or its `package.json` changes. So `harness/contentPolicy.mjs`'s allowlist and the skills range stand.

The engine pin lives in `package.json` and `package-lock.json` only. Its delta touches three `src` files (`dock/Workspace.mjs`, `dock/interaction/RevealOverlay.mjs`, the new `dock/window/ParticipationLifecycle.mjs`) and no SCSS. The engine's own `package.json` changes one devDependency: `cssnano` `^9.0.5` → `^9.1.0` (neomjs/neo#19343). The Institution declares its own `cssnano` (`^9.1.1`), so that doesn't reach it. The Institution's `apps/` never names `Participation` or `crossWindowSortGroup`, so the new default composition does not double-compose here. The reveal-overlay fix is the one cockpit-facing change.

## The Fix

Move the Brain pin to `dev@92122a0` in its three places, and the engine pin to `dev@08ff2a55e6`. Adapt any Institution spec or harness that the new behavior changes.

## Acceptance Criteria

- [ ] AC-1: `package.json`, `package-lock.json` and `ci.yml` name the same Brain commit, and it contains Brain PRs `#705`, `#716`, `#718` and `#720`. `package.json` and the lock name engine `08ff2a55e6`.
- [ ] AC-2: the Institution's suites are green on the new pins. That covers CI's cross-repository contract, the Brain-bound e2e and NL battery run locally (`FleetCockpitTabDragIndicatorsNL` included), and the Darwin visual goldens.
- [ ] AC-3 `[L4-deferred — operator handoff needed]` (post-merge, installed): the repackaged app boots on pin 8, and a Fleet-launched GUI seat arms its wake route (neomjs/neo-agent-brain#705's AC-9). Owner after merge: #7, with the receipt on #12.

## Out of Scope

- neomjs/neo-agent-brain#724 (`setRepo` / `setRepos` refusals as data). It changes `setRepo`'s answer, which `AddAgentFlow` reads, so it rides the next pin with that adaptation and #407.
- The repackage and install (Emmy, #7).

## Related

#402 / #403 (pin 7) · #405 · #7 · #12 · neomjs/neo-agent-brain#571 / #705 · neomjs/neo#19351 / neomjs/neo#19340

## Sweeps

- Live latest-open sweep: the latest 20 open Institution issues at 2026-10-01T19:23:18Z. No pin issue is open (#402 closed with #403).
- A2A: the last 10 messages, all read-states. Emmy (19:22:51Z) asks for one final coherent pin before she builds. Grace (19:22:37Z) leaves pin 8 to "Ada/Vega, whoever takes" it. No claim.
- MC sweep: one query on the problem's nouns (the next pin after `#705`'s wake-route arming), 5 results, no prior decision.
- Own-assignment sweep: 2 open (#407, #408), not overlapping.

Origin Session ID: 84a3bf84-c9cb-4215-818a-d9640f49669a

Authored by Ada (Claude Opus 5.5, Claude Code).


## Timeline

- 2026-10-01T19:23:41Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-01T19:23:41Z @neo-opus-ada added the `enhancement` label
- 2026-10-01T19:23:41Z @neo-opus-ada added the `agent-os` label
- 2026-10-01T19:23:41Z @neo-opus-ada added the `ai` label
- 2026-10-01T19:27:34Z @neo-opus-ada cross-referenced by PR #410
- 2026-10-01T19:29:24Z @neo-opus-ada changed title from **Brain pin 8: the installed FM carries the Fleet's wake-route arming** to **Brain pin 8 and engine pin: the installed FM carries the Fleet's wake-route arming**
- 2026-10-01T19:36:48Z @neo-opus-ada referenced in commit `3588bd0` - "chore(deps): the engine pin moves to dev@08ff2a55e6, and the cockpit spy host sits in no window (#409)

Engine 08ff2a55e6 carries neomjs/neo#19351 (the dock Workspace's default cross-window Participation) and neomjs/neo#19340 (a reveal-overlay focus fix). The refresh's new participation sync reads windowId, which a bare Object.create(prototype) spy host cannot answer, so the projection spec's spy identity declares the window it does not sit in."
- 2026-10-01T19:41:33Z @neo-opus-ada referenced in commit `b60fc60` - "chore(deps): the visual baseline stamp follows the engine pin (#409)

The engine version is a stamped style input. The goldens hold on engine 08ff2a55e6 (visual 27/27, NL captures 53/53, no snapshot updated), so only the stamp moves."
- 2026-10-01T19:58:41Z @neo-gpt-sophie cross-referenced by PR #724
- 2026-10-01T20:04:30Z @tobiu referenced in commit `a252fa6` - "chore(deps): Brain pin 8 (dev@92122a0) and the engine pin (dev@08ff2a55e6) (#409) (#410)

* chore(deps): Brain pin 8 (dev@92122a0) carries the Fleet's wake-route arming (#409)

The pin moves in its three places to Brain dev@92122a0: #705 arms a Fleet-launched GUI seat's wake route as the seat, meeting #405's receiver handoff, plus #716 and the #718 / #720 flake fixes. Nothing under the Brain's src/ or package.json changed since 5041af0.

* chore(deps): the engine pin moves to dev@08ff2a55e6, and the cockpit spy host sits in no window (#409)

Engine 08ff2a55e6 carries neomjs/neo#19351 (the dock Workspace's default cross-window Participation) and neomjs/neo#19340 (a reveal-overlay focus fix). The refresh's new participation sync reads windowId, which a bare Object.create(prototype) spy host cannot answer, so the projection spec's spy identity declares the window it does not sit in.

* chore(deps): the visual baseline stamp follows the engine pin (#409)

The engine version is a stamped style input. The goldens hold on engine 08ff2a55e6 (visual 27/27, NL captures 53/53, no snapshot updated), so only the stamp moves."
- 2026-10-01T20:04:30Z @tobiu closed this issue
- 2026-10-01T20:07:09Z @neo-fable cross-referenced by #12
- 2026-10-01T20:15:08Z @neo-opus-vega cross-referenced by #411

