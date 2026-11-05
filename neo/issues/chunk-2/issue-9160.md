---
id: 9160
title: 'test: Implement Computational Budget assertion in Grid Pooling tests'
state: OPEN
labels:
  - enhancement
  - ai
  - testing
  - performance
  - grid
assignees:
  - neo-gemini-maryam
createdAt: '2026-02-15T01:01:36Z'
updatedAt: '2026-11-05T10:30:09Z'
githubUrl: 'https://github.com/neomjs/neo/issues/9160'
author: tobiu
commentsCount: 5
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
---
# test: Implement Computational Budget assertion in Grid Pooling tests

Recent performance optimizations in the Grid revealed that significant regressions (e.g., 7000+ redundant render calls) can occur silently without affecting the VDOM delta output or breaking functional tests.

**Goal:**
Enhance `test/playwright/unit/grid/Pooling.spec.mjs` to enforce a "Computational Budget".

**Tasks:**
1.  Inject a spy into `Neo.grid.column.Component.prototype.cellRenderer` within the test environment.
2.  Count the number of executions during a controlled vertical scroll operation.
3.  Assert that the execution count stays within an acceptable limit (e.g., < 100 for a 10-row scroll), effectively creating a performance regression guard.

This will catch "silent heaters" where the App Worker performs excessive work that is filtered out before reaching the Main Thread.

## Timeline

### @github-actions - 2026-05-16T04:46:04Z

This issue is stale because it has been open for 90 days with no activity.

### @github-actions - 2026-05-30T05:04:37Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

### @neo-fable-mnemosyne - 2026-10-09T13:03:22Z

**reopen — valid-but-later** (2026-10-09, #19489 set B, grid): the computational budget (count `cellRenderer` executions during a controlled scroll) is not built — `Pooling.spec.mjs` guards structure, `column/Component.spec.mjs` the short-circuit, neither the count. Unowned, no milestone.

🪢 Mnemosyne (Claude Fable 5.1 · Claude Code) · session 2ea2911e-ebbd-49be-9471-3e77369ca2b5

- 2026-10-09T13:03:57Z @neo-fable-mnemosyne cross-referenced by #19489
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-gemini-maryam - 2026-11-05T10:29:54Z

### Ticket Triage & Intake (`#9160`, `2026-11-05`)

- **Triage (`ticket-triage`):** Triaged per `ticket-triage` skill. Applied: `enhancement`, `grid` (alongside existing `ai`, `testing`, `performance`). Stage retrospective passed.
- **7-Day Rule Attribution (`ticket-intake §3a`):** Claimed from `@tobiu` (`updatedAt: 2026-10-09T13:03:25Z`, `now - lastQualifyingActivity = 27d >= 7d`; reopened on `2026-10-09` as unowned backlog).
- **Intake Classification (`ticket-intake §2`):** **Sharpen & Execute**.
- **Code-Reality Verification ("10k+ Commits Mental Model"):**
  1. In `test/playwright/unit/grid/Pooling.spec.mjs` (`beforeEach`, lines 109–133), all 6 default columns (`id`, `name`, `age`, `country`, `email`, `score`) are plain `Neo.grid.column.Base` instances — none is `type: 'component'` (`Neo.grid.column.Component`). Spying on `Neo.grid.column.Component.prototype.cellRenderer` on the default grid without a component column would record `0` calls (`0 < 100`, a vacuous pass).
  2. In `src/grid/column/Component.mjs:44`, component columns default to `hideMode: 'visibility'` (rendered in Pass 2 "Permanent Cells" of `Neo.grid.Row#createVdom`, lines 595–689). During `grid.view.syncBodies(scrollAmount)`, `Neo.grid.Body#createViewData` (lines 884–897) iterates `mountedRows[0]..endIndex` and calls `row.updateContent(...)`, which short-circuits (`src/grid/Row.mjs:807-809`) when `!force && me.record === record && me.rowIndex === rowIndex && !me.hasStaleGeometry()`.
  3. **Sharpened AC & Falsifier:** Configure a pooled grid with a `type: 'component'` column (`Neo.grid.column.Component` rendering a lightweight `Neo.component.Base` cell component), reset the `cellRenderer` counter after initial mount, and assert:
     - Scrolling down 10 rows (`400px`) executes `Neo.grid.column.Component.prototype.cellRenderer` strictly for the 10 newly recycled rows (`executionCount === 10`, asserting `executionCount > 0` for non-vacuity and `executionCount <= 10` / `< 100` for computational budget, with `0` `insertNode`/`removeNode` structural deltas).
     - Re-invoking `createViewData()` at the same scroll offset executes `cellRenderer` `0` times (`Row#updateContent` idempotency guard).

— Maryam (Gemini 4 Argon, Antigravity) · session 70151411-91ec-4496-b8f0-ea3b31ddb60b 🧭

### @neo-gemini-maryam - 2026-11-05T10:30:09Z

**`[lane-override]` reassignment audit-trail** (#11537 §AC8)

**Previous assignees:** `@tobiu`
**New assignees:** `@me`
**Reason:** 7-day rule pickup (now - lastQualifyingActivity = 27d >= 7d; reopened 2026-10-09 as unowned backlog per operator 2026-11-05 backlog directive)

*Audit-trail per AGENTS.md §6.5 — `acknowledgedReassign` reason persistence. Graph-ingested via Retrospective daemon comment-scan path.*

- 2026-11-05T10:35:41Z @neo-gemini-maryam cross-referenced by PR #20299

