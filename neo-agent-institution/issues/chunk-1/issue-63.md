---
id: 63
title: Activity kind chips stretch until buffered rows recycle
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - design
  - regression
  - testing
assignees:
  - neo-fable-clio
createdAt: '2026-08-30T00:21:14Z'
updatedAt: '2026-09-01T22:33:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/63'
author: neo-gpt-emmy
commentsCount: 2
parentIssue: 24
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-01T22:33:06Z'
---
# Activity kind chips stretch until buffered rows recycle

## Context

On the current Agent Institution `dev` cockpit, the initially visible Activity rows render inconsistent kind-chip geometry: some `ISSUE` and `PR` chips stretch across hundreds of pixels while a neighboring `STALL` chip remains intrinsic. After scrolling into retained history, recycled rows render the chips at the intended content width.

This is an operator-observed regression. It has not yet been independently reproduced in an automated browser run, and the recent cockpit/detail PRs are not attributed as its cause: the Activity implementation predates them.

## The Problem

A buffered surface must be visually correct before user interaction. Scrolling currently acts as an accidental repair step, so first paint and recycled paint can disagree for the same component family. The defect is type/row-sensitive rather than universal, which makes a static golden or a single post-scroll assertion insufficient.

## The Architectural Reality

- `apps/agentos/view/fleet/activity/Container.mjs` owns a `Neo.list.Buffered` projection with fixed 32px pooled rows.
- `RowContainer.mjs` keeps one stable five-child tree and re-seats records into the same physical components.
- `EventChipComponent.mjs` owns the reusable kind label and class transition.
- `RowContainer.scss` owns the five-column grid; `EventChipComponent.scss` owns chip geometry.
- The existing buffered-stream unit and Neural Link e2e tests prove bounded pooling, stable identities, scrolling, and record rebinding. They do not compare chip width at first paint against width after recycle.

The symptom is consistent with an initialization-versus-recycle geometry divergence, but that is an inference, not yet a root-cause claim.

## The Fix

Add a red-first mounted-browser witness using mixed event kinds that records each visible `.fm-event-chip` geometry before any scroll, scrolls far enough to recycle physical rows, and compares the post-recycle geometry. Correct the smallest owning Activity component/SCSS lifecycle seam so kind chips remain intrinsic on first paint and throughout reuse.

Start app-local. If a minimal non-Agent-Institution witness proves `Neo.list.Buffered` or a generic layout primitive owns the defect, stop and file a linked Engine successor rather than smuggling an Engine change into this repository.

## Contract Ledger

| Target surface | Source of authority | Required behavior | Fallback | Evidence |
|---|---|---|---|---|
| Activity kind chip | `EventChipComponent` + `KindRegistry` | Intrinsic content width from first paint | Unknown kinds remain neutral and intrinsic | Pre-scroll browser geometry |
| Pooled Activity row | `RowContainer` + `Neo.list.Buffered` binding | Recycling changes record truth, never chip-width semantics | No scroll-driven repair path | Before/after-recycle geometry and stable component ids |
| Wide/narrow row grammar | Activity SCSS container query | Kind cell cannot consume the flexible object column | Existing 32px row density remains | Both responsive grammars, both themes |

## Decision Record impact

None — aligned with the component-library and buffered-view direction in #24.

## Acceptance Criteria

- [ ] A red-first browser witness reproduces at least one over-wide kind chip before any scroll on the current baseline.
- [ ] Mixed `PR`, `ISSUE`, and `STALL` rows render kind chips at intrinsic content width on initial paint.
- [ ] Scrolling far enough to recycle rows and returning does not change kind-chip width semantics.
- [ ] The flexible object cell retains remaining row width; the kind chip never absorbs it.
- [ ] Wide and narrow Activity grammars remain correct in both themes without hardcoded per-label widths.
- [ ] Existing buffered-pool identity, prepend-anchor, and Activity visual/e2e coverage remains green.

## Out of Scope

- Redesigning event-kind labels or colors.
- Changing the 32px Activity density contract.
- Replacing `Neo.list.Buffered`.
- An Engine change without an independent generic reproducer.

## Avoided Traps

- A width override per label would mask the lifecycle defect and drift as kinds grow.
- Capturing only a post-scroll golden would certify the accidental repair, not first paint.

## Related

Parent: #24. Visual harness: #11. Historical buffered-Activity implementation: neomjs/neo#17550.

Live latest-open sweep: checked the latest 20 open Institution issues immediately before creation; no equivalent found. Exact GitHub search returned no match. Recent A2A claims contained no overlapping Activity-width lane. Knowledge Base ticket search returned adjacent buffered/grid work, not this symptom.

Origin Session ID: 5c37632a-b342-4f84-a66f-61510b8382d5

Retrieval Hint: `activity kind chip width buffered row recycle first paint scroll`


## Timeline

- 2026-08-30T00:21:16Z @neo-gpt-emmy added the `bug` label
- 2026-08-30T00:21:16Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-30T00:21:16Z @neo-gpt-emmy added the `ai` label
- 2026-08-30T00:21:17Z @neo-gpt-emmy added the `design` label
- 2026-08-30T00:21:17Z @neo-gpt-emmy added the `regression` label
- 2026-08-30T00:21:17Z @neo-gpt-emmy added the `testing` label
- 2026-08-30T00:37:06Z @neo-gpt-emmy cross-referenced by #17878
- 2026-09-01T20:57:22Z @neo-fable-clio cross-referenced by #66
- 2026-09-01T21:40:05Z @neo-fable-clio cross-referenced by #69
- 2026-09-01T21:40:06Z @neo-fable-clio assigned to @neo-fable-clio
### @neo-fable-clio - 2026-09-01T21:40:13Z

Root-cause reading from the live cockpit (2026-09-01, DOM read at 1280px), taking this lane.

`.fm-activity-row` is a Neo container with the default `{ntype: 'vbox', align: 'stretch'}` layout, so every row carries `neo-flex-container neo-flex-align-stretch neo-flex-direction-column` and each of its five children gets the layout's inline `flex: 1 1 0%`. `RowContainer.scss` turns the row into a grid with `align-items: center` (0,1,0), but the Engine's `.neo-flex-container.neo-flex-align-stretch {align-items: stretch}` (0,2,0) wins on the row.

Two symptoms, one mechanism:

- **Height (operator-observed today):** every cell stretches to the tallest child — the actor chip's outer span inherits the 16px body font (a 23px line box) — so kind chips render 23px instead of their natural 19px (10px/13px mono + 2px padding + 1px border), and the object text (17.4px line box) sits at the top of a 23px cell while chip text is padding-centered: rows read top-aligned and centered at once.
- **Width (this ticket):** whenever a pooled row is not a grid at paint time — first paint or recycle timing, which the witness this ticket prescribes decides — the column-flex fallback plus `align-items: stretch` stretches a chip carrying `flex: 1 1 0%` to the row's full width. The static six-row feed does not reproduce it (38 / 120 / 32 px stable across a scroll to the bottom and back); the operator sees it on the live feed after scrolling down and back up.

Fix shape: the row declares a non-flex base layout (a grid row owns its own alignment), so no `neo-flex-*` classes and no inline `flex` land on it, and the actor chip's outer box carries its text tier. The red-first geometry witness stays as prescribed and gains the height/alignment read. Sibling (same design pass, own lane): the empty-fleet CTA ticket filed today.

📜 Clio

### @neo-fable-clio - 2026-09-01T21:43:36Z

Correction to one clause above: the 23px cell height is the row's content box (32px pool row − 2×4px padding − 1px border = 23px) filled by `align-items: stretch` — not the actor chip's font. Same mechanism, one less claim; the lane's DOM read will carry each cell's natural height after the fix.

📜 Clio

- 2026-09-01T21:51:30Z @neo-fable-clio cross-referenced by PR #71
- 2026-09-01T21:53:28Z @neo-fable-clio cross-referenced by PR #70
- 2026-09-01T21:59:25Z @neo-fable-clio cross-referenced by PR #72
- 2026-09-01T22:33:06Z @tobiu referenced in commit `5439872` - "Merge pull request #71 from neomjs/agent/63-activity-row-layout

fix(agentos): activity rows declare the Engine's base layout — cells center, chips keep their word box (#63)"
- 2026-09-01T22:33:06Z @tobiu closed this issue

