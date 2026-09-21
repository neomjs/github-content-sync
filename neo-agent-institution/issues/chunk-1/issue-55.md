---
id: 55
title: 'Mechanical file-size guard: the apps/** product surface holds the 1k-LOC bar'
state: CLOSED
labels: []
assignees:
  - tobiu
createdAt: '2026-08-29T22:13:02Z'
updatedAt: '2026-08-29T22:41:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/55'
author: tobiu
commentsCount: 0
parentIssue: 22
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-29T22:41:26Z'
---
# Mechanical file-size guard: the apps/** product surface holds the 1k-LOC bar

## Problem Scope

The #22 epic's extraction subs are DONE (#48 → PR #49, #50 → PR #52): `FleetCockpit.mjs` no longer exists, and the largest product app files sit under the bar — measured at dev head today: `cockpit/Controller.mjs` 928 · `cockpit/VesselContainer.mjs` 902 · `cockpit/Container.mjs` 899 · `cockpit/LivenessController.mjs` 880 LOC.

What is missing is the epic's own closing clause: **"the invariant gets teeth"** — a mechanical file-size guard, because this debt class regressed exactly once before precisely because no gate watched it (3,327 → 3,487 LOC across one feature PR, per the epic's measurements). Without a gate, the next feature wave regrows the monolith silently.

## Intended Solution

`buildScripts/checkAppFileSizes.mjs`, following the existing `check-*` family pattern (`checkVisualBaselines.mjs`: pure exported core + thin CLI):

- **Scope:** `apps/agentos/**/*.mjs` — the PRODUCT surface only. `apps/agentos/childapps/**` is explicitly EXEMPT with rationale in the script header: neomjs/neo#16322 owns the demo relocation out of the product app and neomjs/neo#15614 owns DemoBWorkspace's decomposition — the epic's out-of-scope rule, made visible where the guard runs (the exemption retires when #16322 lands).
- **Ladder:** warn ≥ 900 LOC (prints, exit 0) · error > 1,000 LOC (prints offenders, exit 1) — the epic's warn → error ladder in one script, thresholds as named constants.
- **Wiring:** npm script `check-app-file-sizes` + a step in the Isolated Institution CI job beside `check-visual-baselines`.
- **Witness:** a unit spec in `test/playwright/unit/buildScripts/` driving the pure core over fixture listings — over-budget file → error row; warn-band file → warn row, exit-clean; childapps path over budget → exempt; the live tree passes.

## Acceptance Criteria

- [ ] `checkAppFileSizes.mjs` exists with a pure, unit-drivable core; thresholds are named constants (900 warn / 1,000 error).
- [ ] ~~`childapps/**` exemption is in code WITH the #16322/#15614 rationale comment.~~ **Amended 2026-08-30 (measurement at implementation):** dockdemo/DemoBWorkspace is ALREADY relocated — tracked `childapps/**` totals 39 LOC across 2 files — so the guard ships ONE budget with NO exemption (a carve would exempt nothing today and hide a returning demo monolith tomorrow); the decision + rationale live in the script header, per the epic's "made visibly" clause.
- [ ] npm script wired; Isolated Institution CI runs it.
- [ ] Unit witness covers the ladder arms (error/warn/pass, boundary-exact) AND the hermetic inventory contract over a throwaway git repo: app subtrees included (childapps carve-free — an over-budget childapps module ERRORS), outside modules excluded, `wc -l` edge cases pinned. (Amended 2026-08-30 with the no-exemption decision + reviewer scope-falsifier: a narrowed pathspec or reintroduced carve must red.)
- [ ] The current tree passes the guard (proof the bar holds at adoption).

## Out of Scope

Further extraction work (done via #48/#50) · engine-side checks (`check-reactive-tags` is engine-delivered) · dockdemo decomposition/relocation (neomjs/neo#15614 / neomjs/neo#16322) · repairing the dead `check-data-tracking` npm entry (`buildScripts/checkDataTracking.mjs` does not exist — noticed during this sweep, recorded here for a future micro-lane, not absorbed).

## Related

Parent: #22 (this is its named mechanical-guard sub, the last open clause). Reference shape: `buildScripts/checkVisualBaselines.mjs` + its spec.

Authored by Clio (Fable 5, Claude Code). Origin Session ID: 729e92a3-fa8a-4c4f-bc72-6c0ad1a5e604

Retrieval Hint: `query_raw_memories("file size guard 1k LOC bar check-app-file-sizes agentos")`

Creation notes: live latest-open sweep run this turn (30 open issues listed; keyword sweeps "guard file size" / "check lint LOC" — no equivalent, only parent #22). A2A window clean.




## Timeline

- 2026-08-29T22:13:10Z @tobiu assigned to @tobiu
- 2026-08-29T22:16:42Z @tobiu cross-referenced by PR #56
- 2026-08-29T22:34:33Z @neo-fable-clio referenced in commit `95cfad3` - "test(build): hermetic inventory witness + exact wc-l and entry guard (#55)"
- 2026-08-29T22:41:26Z @tobiu referenced in commit `d2238f0` - "Merge pull request #56 from neomjs/agent/55-file-size-guard

feat(build): mechanical 1k-LOC app-file guard closes the #22 epic (#55)"
- 2026-08-29T22:41:26Z @tobiu closed this issue
- 2026-08-29T22:43:57Z @neo-fable-clio cross-referenced by #22

