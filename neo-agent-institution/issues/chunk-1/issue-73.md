---
id: 73
title: 'The Neural Link battery''s out-of-glob witnesses: four stated reds and the membership glob'
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - regression
  - testing
assignees:
  - neo-fable-clio
createdAt: '2026-09-01T22:53:44Z'
updatedAt: '2026-09-02T12:01:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/73'
author: neo-fable-clio
commentsCount: 1
parentIssue: 10
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-02T12:01:19Z'
---
# The Neural Link battery's out-of-glob witnesses: four stated reds and the membership glob

Sub-issue of #10 (the cockpit's proof surface). Follow-up of #66 / PR #70, where these four were disclosed as stated reds and deliberately not masked.

## Context

PR #70 repaired the nine `FleetCockpit*` witnesses and gave the battery a script — `npm run test-e2e:nl` = `playwright test -c test/playwright/playwright.config.e2e.mjs agentos/FleetCockpit --workers=1`. That glob is a filename prefix: it reaches sixteen specs and excludes every Neural Link witness whose name does not start with `FleetCockpit`, so a spec outside it can rot, or be repaired, with nothing announcing either (Ada's `[KB_GAP]` on PR #70: *"the glob is a membership decision with no observer"*). Running the out-of-glob specs on the #70 head (2026-09-01) produced four honest reds and one green (`OperatorMailboxNL`):

| spec | stops at | reading |
|---|---|---|
| `FleetGridKeyboardA11y` | `ArrowDown moves native list-item focus` — focus stays on `neo-fm-fleet-roster-list-1__1` | product-or-witness: the roster list's keyboard navigation after the #58/#60 rebuilds — triage first, a product fix if the list's Navigator no longer moves focus |
| `AgentCardSynthesisRenderNL` | `agentcard-synthesis-dark-narrow-294` golden: expected 314×407, received 314×405 (0.01 diff) — identical on clean `dev`, verified by running it on the unmodified branch | a 2px shorter card at 294px; needs a design read of the 2px before any re-capture (a re-captured golden blesses what it shows — PR #71's RA-1) |
| `FleetCatchUpNL` | `toContainText("Caught up through Jul 18 02:00 PM")` against the rendered `18 Jul 12:00` | the assertion encodes its author's machine clock and locale; the ViewerTime ladder renders the viewer's — the burst spec's comment already names this trap and the honest shape (a format regex + the wire instant on `title`) |
| `FleetMemoriesNL` | `toContainText("Pick an agent to read their recent sessions.")` against the rendered `Select an agent card in the roster to read their recent sessions.` | retired copy; assert the current words, or the element's role, not a sentence that moved |

## The Problem

A witness battery whose membership is a prefix cannot say what it does not cover, and four of the cockpit's rendered-behaviour contracts (keyboard roster navigation, the card at vessel width, the catch-up window, the memories tab) currently have no green anywhere. The reds are honest (each stops at a real assertion) but they are also silent: no script runs them, so the next Engine-pin bump or chrome rebuild rots them further without a signal.

## The Architectural Reality

- `package.json` `test-e2e:nl` (glob `agentos/FleetCockpit`); `README.md` "Brain-connected maintainer workflow" names the battery and its `-- --headed` receipt.
- `test/playwright/playwright.config.e2e.mjs` ignores every `neuralLink` spec unless `NEO_AGENTOS_RUNTIME_ROOT` is an absolute path; the CI `cross-repository` job only `--list`s — Institution CI never executes a Neural Link witness (the designed Brain split, stated in #66). The battery is a local receipt by construction.
- `apps/agentos/view/fleet/roster/List.mjs` + `SelectionModel.mjs` own the keyboard contract the A11y witness reads; `apps/agentos/view/fleet/catchup/**` and `memories/**` own the two copy/format seams; the AgentCard golden reads `apps/agentos/view/fleet/roster/card/**` at 294px.
- Precedent for the time assertion: `FleetActivityStreamBurstNL` asserts `/\d{2}:\d{2}/` on the rendered text and the exact wire instant on `title`, because the fixture's browser context does not receive the config's locale/timezone emulation.

## The Fix

- Triage the A11y red before touching the spec: drive ArrowDown on the live roster and read `document.activeElement`; if the list's Navigator no longer moves item focus, that is a product regression and the fix lands in the list, the witness stays as the falsifier. If focus moves and the witness reads the wrong node, repair the read.
- Read the 2px on the AgentCard golden (what shrank, whether it is the family rail, the telltale line or padding), decide, then re-capture — the design read is the deliverable, the re-capture is its receipt.
- CatchUp: assert the rendered window through the ViewerTime shape (a format regex) and pin the wire instant on the element's `title`, the burst precedent. Memories: assert the current empty-state copy.
- Widen the script once the four are green: `test-e2e:nl` runs every Neural Link witness under `test/playwright/e2e/agentos/` (a path regex on the `NL` suffix, not a prefix), and the README paragraph names every residual that stays red with its owner, so a red there reads as a known, owned red — not the glob.

## Acceptance Criteria

- [ ] `FleetGridKeyboardA11y` green, with the triage outcome recorded in the PR (product fix or witness repair, and why).
- [ ] `AgentCardSynthesisRenderNL` green after a design read of the 2px, recorded in the PR before the re-capture.
- [ ] `FleetCatchUpNL` and `FleetMemoriesNL` green with assertions that survive the viewer's locale and the current copy.
- [ ] `npm run test-e2e:nl` reaches every Neural Link witness under `test/playwright/e2e/agentos/`; the battery on the PR head is green except the stated reds, each owned by an open ticket the README names (amended 2026-09-02 on the reviewer's ask: the widening found `AddAgentJourneyNL` → #74 and `FleetGridScaleNL` → #78; the Engine FLIP-settle race on `FleetCockpitDockNL` and the rail-drawer witness is occasional, not a standing red — measured 2/2 green headless on 2026-09-02).
- [ ] CI unit/components/isolated e2e/visual stay green; no golden re-captured without its design read in the body.

## Out of Scope

- The Engine's FLIP-settle race (`FleetCockpitDockNL`, the rail-drawer witness) — occasional, defect-noted on #65; promoted to an Engine ticket only on a reproducible hold.
- Making Institution CI execute Neural Link witnesses — #64's CI wiring and the Brain split own that.
- The design-pass residuals ledgered on #10 (314px shell-title collision, the `unobse…` clip, the light-theme shell title) — the design-pass leaf.
- The two reds the widening found: `AddAgentJourneyNL` (#74) and `FleetGridScaleNL` (#78) — their own leaves.

## Related

Parent: #10. Origin: #66 / PR #70 (the disclosure), PR #71 (the golden-blessing rule). Siblings: #64 (CI baseline), #11 (visual harness), #74 and #78 (the reds the widening found).

Live latest-open sweep: checked the latest 20 open Institution issues at 2026-09-01T22:55Z — #64 (CI baseline) and #11 (visual harness) are adjacent, neither names these specs; A2A lane-claims over the last 30 messages cover Engine dock leaves, skills and Brain — none on this scope; Memory Core raw query on the four spec names returned nothing.

Origin Session ID: f353cda0-1b36-49e4-9b73-3f7aec1896e0

Retrieval Hint: `neural link battery out-of-glob witnesses A11y ArrowDown AgentCard 2px golden CatchUp ViewerTime Memories copy test-e2e:nl glob`

📜 Clio


## Timeline

- 2026-09-01T22:53:45Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-01T22:53:46Z @neo-fable-clio added the `bug` label
- 2026-09-01T22:53:46Z @neo-fable-clio added the `agent-os` label
- 2026-09-01T22:53:46Z @neo-fable-clio added the `ai` label
- 2026-09-01T22:53:46Z @neo-fable-clio added the `regression` label
- 2026-09-01T22:53:47Z @neo-fable-clio added the `testing` label
### @neo-fable-clio - 2026-09-01T23:03:32Z

**Item 5, found while this lane was open (2026-09-01 22:53Z):** `FleetCockpitRailDrawerErgonomicsNL` runs in the Institution CI's *isolated* e2e job (it carries no `neuralLink` fixture despite its name) and went red there once on PR #72's head `c1162f7` at "dismissal clears the mark" — after Escape the rail tab kept `pressed` — while the same head passes the spec 3/3 locally in isolated mode and the CI rerun on the next head went green. Earlier tonight the same spec read an empty reveal title once in a five-spec headed run and was green alone. Both are the Engine's FLIP-settle race (the DockNL #6 family) reaching this witness through the rail tab's mark. Disposition for this lane: name it beside the DockNL hold in the README (a red there is the Engine race, not the battery); if the isolated CI job keeps tripping on it, the witness gains a settle-aware wait for the mark, not a retry.

📜 Clio

- 2026-09-01T23:24:21Z @neo-fable-clio cross-referenced by #74
- 2026-09-01T23:30:31Z @neo-fable-clio cross-referenced by PR #75
- 2026-09-01T23:32:33Z @neo-fable-clio referenced in commit `69a5586` - "chore(test): merge dev after #72 and refresh the visual baseline stamp (#73)"
- 2026-09-02T00:24:51Z @neo-fable-clio referenced in commit `013caf3` - "test(agentos): the rail-drawer witness joins the Brain-root battery — its Escape dismissal is the Engine settle (#73)

The isolated CI job ran this dock witness by accident of the neuralLink ignore rule (the file used no fixture) and tripped on the reveal machine's FLIP settle — 2 of 4 runs, then a deterministic timeout of the settle-aware wait on the CI runner while the same head passes 3/3 headless locally. The witness now reads the cockpit through the Neural Link fixture, which gates it behind the Brain root like every other dock witness; the headed battery is its receipt, and the README says so. Isolated e2e: 8 passed without it; under the Brain root: green."
- 2026-09-02T00:49:36Z @neo-fable-clio cross-referenced by PR #77
- 2026-09-02T09:29:15Z @neo-fable-clio cross-referenced by #78
- 2026-09-02T09:35:48Z @neo-fable-clio referenced in commit `691e96c` - "test(agentos): the golden bar loosens only under a headed run, and every stated red names its owner (#73)"
- 2026-09-02T09:48:39Z @neo-fable-clio referenced in commit `7e0d14e` - "chore(test): refresh the visual baseline stamp after the golden-bar edit (#73)"
- 2026-09-02T10:44:53Z @neo-fable-clio referenced in commit `7d5ad99` - "test(agentos): the headless golden compare stays byte-exact; only a headed run carries the antialiasing allowance (#73)"
- 2026-09-02T12:01:20Z @tobiu referenced in commit `c0f4ed5` - "Merge pull request #75 from neomjs/agent/73-battery-out-of-glob

test(agentos): the battery reaches every Neural Link witness; the stated reds repaired at their seams (#73)"
- 2026-09-02T12:01:20Z @tobiu closed this issue
- 2026-09-04T11:21:52Z @neo-fable-clio cross-referenced by PR #95

