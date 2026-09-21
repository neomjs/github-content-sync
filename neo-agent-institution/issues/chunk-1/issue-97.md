---
id: 97
title: Split the 3095-line cockpit container spec by concern — the one FM file the LOC bar reads red
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - refactoring
  - testing
assignees:
  - neo-fable-clio
createdAt: '2026-09-04T12:46:44Z'
updatedAt: '2026-09-04T15:59:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/97'
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
closedAt: '2026-09-04T15:59:59Z'
---
# Split the 3095-line cockpit container spec by concern — the one FM file the LOC bar reads red

## Context

The operator's file-size bar for this codebase, stated 2026-09-04: up to 1k lines green, 1.5k yellow, 2k red — `component.Base` itself stays under 2k. A sweep of the Fleet Manager on the same day: every one of the 102 source files under `apps/agentos` is ≤ 1000 lines (two sit at the line: `view/fleet/cockpit/Controller.mjs` 995, `view/fleet/cockpit/Container.mjs` 995), and exactly one file reads red — `test/playwright/unit/apps/agentos/view/fleet/cockpit/container.spec.mjs` at **3095 lines, 108 arms**. `Accounts.spec.mjs` (1138) and `harness/main.mjs` (1472) are the yellow band; not this ticket.

Live latest-open sweep: checked the latest 20 open issues at 2026-09-04T12:44Z; no equivalent found. A2A claim sweep (30 most recent messages, all read states): no claim on this file. Memory Core rationale sweep: no prior decision on splitting this spec.

## The Problem

The container spec is the cockpit's catch-all: ten top-level `describe` blocks that are ten separate concerns, accreted over #14611, #14868, #14978, #15293 and #15377 — activity-feed binding, the Store-backed roster, whole-fleet control, the settled-intent re-poll, the spine-banner pipeline, the liveness owner lifecycle, the wake-routes read, the tasks read, the operator mailbox, the operator-seat identity posture. Its siblings already took the shape the operator asks for — `projection.spec.mjs`, `tearOut.spec.mjs`, `popOut.spec.mjs`, `residentBoot.spec.mjs`, `spineBanner.spec.mjs`, `custodyHeal.spec.mjs` are each one concern, 150–860 lines. Nobody can navigate 3095 lines to find the arm that owns a behavior, and every new cockpit contract lands in the catch-all by default because it is the file that exists.

## The Architectural Reality

- The file is one `setup({appConfig})` + ten `test.describe` blocks (lines 126, 262, 973, 1222, 1476, 2089, 2536, 2618, 2722, 3044), each with its own `beforeEach` cockpit construction — the blocks share the harness shape, not state.
- The unit tier's Playwright config (`test/playwright/playwright.config.unit.mjs`) picks up every `*.spec.mjs` under `test/playwright/unit`; a split needs no config change.
- The cockpit specs that already stand alone use the same imports and the same `Neo.create(FleetCockpit, {stateProvider: {module: CockpitStateProvider, stores: {…}}})` boot (`residentBoot.spec.mjs` lines 1–45 is the template).

## The Fix

Split `container.spec.mjs` by its own describe boundaries into concern-named siblings in the same directory, moving arms verbatim (no assertion changes, no arm renames): `activityFeed.spec.mjs`, `rosterStore.spec.mjs`, `fleetControl.spec.mjs`, `intentRepoll.spec.mjs`, `spineBannerPipeline.spec.mjs` (or fold into the existing `spineBanner.spec.mjs` if the harness matches), `livenessLifecycle.spec.mjs`, `wakeRoutesRead.spec.mjs`, `tasksRead.spec.mjs`, `operatorMailbox.spec.mjs`, `operatorSeatIdentity.spec.mjs`. Each new file carries the file-level JSDoc its block already has and its own `setup({appConfig: {name}})` with a distinct app name. `container.spec.mjs` disappears, or keeps only what is genuinely about the Container class itself if such arms exist (state that in the PR). Arm count before and after must match (108), and the full unit tier stays green.

## Acceptance Criteria

- [ ] AC-1 No file under `test/playwright/unit/apps/agentos/view/fleet/cockpit/` exceeds 1000 lines; every describe block of the former catch-all lives in a sibling named for its concern.
- [ ] AC-2 The arm count is unchanged (108 moved, 0 rewritten) — stated in the PR with the before/after `grep -c "^\s*test("` numbers — and `npm run test-unit` is green.
- [ ] AC-3 Each new file has a file-level JSDoc naming the concern and the contract it pins — moved from the block where the block carried one (eight of ten); the two blocks that never had one (whole-fleet control, operator-seat identity posture) get a JSDoc written from their arm titles, stated in the PR — and a distinct `setup` app name. *(Truth-synced 2026-09-04 from the measured eight-moved / two-added contract, PR #104 review round 1.)*

## Out of Scope

- Rewriting or tightening any arm — a pure move; assertion changes are their own tickets.
- `Accounts.spec.mjs` (1138) and `harness/main.mjs` (1472) — yellow, separate leaves if the bar hardens.
- The two 995-line source files — a growth freeze, not a split; they get their own leaf the day one needs a line.

## Related

#10 (parent arc) · #62 (the newest sibling spec, `custodyHeal.spec.mjs`, filed one concern per file) · the operator's LOC bar, 2026-09-04, in-session.

Ownership: unowned — a one-PR, test-only move any peer can take; not claimed by the author.

Origin Session ID: e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3

Retrieval Hint: `query_raw_memories("cockpit container.spec split by concern 3095 lines LOC bar")`

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3



## Timeline

- 2026-09-04T12:46:46Z @neo-fable-clio added the `enhancement` label
- 2026-09-04T12:46:46Z @neo-fable-clio added the `agent-os` label
- 2026-09-04T12:46:47Z @neo-fable-clio added the `ai` label
- 2026-09-04T12:46:47Z @neo-fable-clio added the `refactoring` label
- 2026-09-04T12:46:47Z @neo-fable-clio added the `testing` label
- 2026-09-04T15:27:38Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-04T15:34:11Z @neo-fable-clio cross-referenced by PR #104
- 2026-09-04T15:59:59Z @tobiu referenced in commit `da4657c` - "Merge pull request #104 from neomjs/agent/97-cockpit-spec-split

test(agentos): the cockpit container spec splits into ten concern-named siblings; the shared fakes move to one module (#97)"
- 2026-09-04T16:00:00Z @tobiu closed this issue

