---
id: 658
title: The setup card's tests read the recipe's step count from the recipe
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - testing
assignees:
  - neo-fable-clio
createdAt: '2026-10-10T14:07:26Z'
updatedAt: '2026-10-10T15:03:36Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/658'
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
closedAt: '2026-10-10T15:03:36Z'
---
# The setup card's tests read the recipe's step count from the recipe

## Context

Since Brain #960 (merged 2026-10-10, "the first-run recipe registers the plane's forge connection as its own row") the first-run recipe has thirteen steps. The Institution tracks the Brain at `dev` in CI (`resolve-org-dev`, #650), and its setup-card tests pin the old count as a literal: every Institution pull request opened today fails the two required checks (`Isolated Institution`, `Explicit Brain contract`) on `createContainer.spec.mjs:145` (`Expected: 12 · Received: 13`) and `:275` (`… · 5 of 12` vs `5 of 13`); the Darwin visual arm `FleetCockpitVisual.spec.mjs:1819` and the e2e `FleetSetupCard.spec.mjs` carry the same literal. Observed on PR #656 (14:02Z) and in a local full visual run at Brain `be7181ba` / Engine `6c1e478f`; `dev`'s last CI run (af1b85d6, 23:09Z) predates the Brain merge, so `dev` is red latently, not visibly.

Design authority: the Institution's own rule that the Brain is the recipe's author (the COLD fixture is evaluated live through the broker, `createContainer.spec.mjs:70`) — only the literal pins are stale.

## The Problem

The fixtures already read the recipe (`COLD = (await HOST.broker.evaluate(TRUSTED, {})).evaluation`); the assertions then compare against the number 12 and strings built from it (`5 of 12`, `2 of 12 observed ok · next: preset`, `12 of 12 observed ok · complete`). A recipe row added in the Brain turns every one of them red without any Institution behavior changing.

## The Fix

Derive, never pin: the step count is `COLD.steps.length` (the unit spec) or the Brain recipe's own `RECIPE_STEPS.length` (`node_modules/neo-agent-brain/ai/services/fleet/firstRunRecipe.mjs`, imported where the fixture cannot be evaluated), and the progress strings are built from it — `${ok} of ${total} observed ok · next: ${id}` with `ok` as the index of the named next step in the recipe's order, `${total} of ${total} observed ok · complete` at the end. Files: `test/playwright/unit/apps/agentos/view/setup/createContainer.spec.mjs` (lines 145, 275, 597, 675), `test/playwright/visual/FleetCockpitVisual.spec.mjs:1819`, `test/playwright/e2e/agentos/FleetSetupCard.spec.mjs` (60, 69, 104, 115, 140, 147). If the Create door's golden changed with the thirteenth row, it is re-captured from a full run and the stamp re-taken.

## Acceptance Criteria

- [ ] AC-1 — unit: `createContainer.spec.mjs` passes at Brain `dev` with no literal step count; the count and the progress strings come from the recipe.
- [ ] AC-2 — the visual arm and the e2e spec derive their strings the same way; the e2e passes locally; the visual run's Create-door arm is green at this head (golden re-captured only if the row changed its pixels), stamp re-taken.
- [ ] AC-3 — CI's two required checks are green on this PR, and PR #656 (re-run at the merged head) turns green without a change of its own.

## Out of Scope

The recipe's content and the thirteenth row's words (Brain #960); the two-exits witness arm (`:1880`, the #613 red); row 1's installed walkthrough (#534).

## Related

Brain #960 (the row), #650 (tracking the Brain at dev), #534 / #351 (row 1), PR #656 (the first PR it blocked).

Sweeps: live latest-20 open Institution issues read 2026-10-10 14:06:33Z — no equivalent (#657 is the dependency alerts); A2A last 30 — no claim on the recipe count; Memory Core — Vega's #960 record names the new row, nothing on the Institution's pins; own assignments (#505 #507 #351 #654 #655) — none on this surface; structure map: test-only, no new file.

Retrieval Hint: "setup card recipe step count 12 13 createContainer.spec resolve-org-dev drift"

Origin Session ID: f45d36fd-6e77-4c89-bd56-dd49d95b5b0a

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session f45d36fd-6e77-4c89-bd56-dd49d95b5b0a

## Timeline

- 2026-10-10T14:07:26Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-10T14:07:28Z @neo-fable-clio added the `bug` label
- 2026-10-10T14:07:28Z @neo-fable-clio added the `agent-os` label
- 2026-10-10T14:07:29Z @neo-fable-clio added the `ai` label
- 2026-10-10T14:07:29Z @neo-fable-clio added the `testing` label
- 2026-10-10T14:34:00Z @neo-fable-clio cross-referenced by PR #659
- 2026-10-10T14:52:37Z @neo-fable-clio referenced in commit `ca9c96f` - "test(setup): the cold served-plane row's status reads by its recipe place, like its reason (#658)"
- 2026-10-10T15:03:36Z @tobiu referenced in commit `bfe4def` - "test(setup): the setup card's tests read the recipe's step count from the recipe (#658) (#659)

* test(setup): the setup card's tests read the recipe's step count from the recipe (#658)

Brain #960 gave the first-run recipe a thirteenth row, register-forge, and
the Institution tracks the Brain at dev: every pull request failed the two
required checks on the setup card's literal "12". The unit, e2e and visual
specs now derive the count, the progress strings and the decision count from
the recipe (COLD.steps / RECIPE_STEPS), their scripted walks run the new
effect, and the pinned setup host scripts the plane's forge-connection
registry — the canonical status answer and the init/register mutations — so
the row's own observer and effect run their real code against it. A plane
that already runs when the host is scripted adopts its binding.

* test(setup): the cold served-plane row's status reads by its recipe place, like its reason (#658)"
- 2026-10-10T15:03:37Z @tobiu closed this issue

