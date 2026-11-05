---
id: 9404
title: '[Epic] Tree Grid & Hierarchical Data Support'
state: OPEN
labels:
  - enhancement
  - epic
  - no auto close
  - ai
  - architecture
  - grid
assignees:
  - tobiu
createdAt: '2026-03-09T10:43:40Z'
updatedAt: '2026-11-05T10:41:29Z'
githubUrl: 'https://github.com/neomjs/neo/issues/9404'
author: tobiu
commentsCount: 3
parentIssue: null
subIssues:
  - '[x] 9405 Create `Neo.data.TreeModel`'
  - '[x] 9406 Create `Neo.data.TreeStore`'
  - '[x] 9407 Create `Neo.grid.column.Tree` & Cell Component'
  - '[x] 9408 Grid Core Integration & TreeGrid Accessibility'
  - '[x] 9409 CSS Animations for TreeGrid Expand/Collapse'
  - '[x] 9410 TreeGrid Documentation & Examples'
  - '[x] 9411 TreeGrid Unit Tests (Data & Logic)'
  - '[x] 9412 TreeGrid Component Tests (UI & Interactions)'
  - '[x] 9413 Create Async Subtree Loading for `Neo.data.TreeStore`'
  - '[x] 9414 Refactor `Neo.data.Store` to unify Record Hydration'
  - '[x] 9415 Support "Turbo Mode" in `Neo.data.TreeStore`'
  - '[x] 9416 Add Error State & Events for Async Tree Loading'
  - '[x] 9417 Optimize `TreeStore` Hot Paths (Performance)'
  - '[x] 9422 Update TreeStore to manage ARIA sibling fields on mutations'
  - '[x] 9423 TreeStore Full CRUD Support & Structural Mutations'
  - '[x] 9425 Refactor Tree cell component architecture for performance'
  - '[x] 9426 Refactor Tree column and component reactivity'
  - '[x] 9427 Fix TreeStore projection mutations and Row VDOM recycling'
  - '[x] 9428 TreeStore: Implement hierarchical sorting for doSort override'
  - '[x] 9429 TreeStore: Implement ancestor-aware filtering for filter override'
  - '[x] 9430 TreeModel: Introduce childCount to decouple isLeaf state from emptiness'
  - '[x] 9431 TreeStore: Fix ARIA desync (siblingIndex and siblingCount) after sort and filter'
  - '[x] 9432 TreeStore: Override clear() to prevent memory leaks and split-brain states'
  - '[x] 9433 TreeStore: Implement bulk expandAll() and collapseAll() methods'
  - '[x] 9434 TreeStore: Decouple clearFilters() from legacy allItems pattern'
  - '[x] 9435 TreeStore: Fix visible projection for dynamic child additions to expanded parents'
  - '[x] 9437 TreeStore: Optimize #allRecordsMap iteration loops'
  - '[x] 9438 TreeStore: Reduce GC pressure and redundant iterations'
  - '[x] 9439 TreeStore: Apply "Anchor & Echo" JSDoc strategy for AI discoverability'
  - '[x] 9445 TreeGrid Documentation: Create high-level architectural guide for TreeStore'
  - '[x] 9447 TreeGrid: Fix 7-click expand/collapse bug and redundant change events'
  - '[x] 9448 TreeGrid Component pooling accumulates `is-leaf` class leading to visual bugs'
subIssuesCompleted: 32
subIssuesTotal: 32
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# [Epic] Tree Grid & Hierarchical Data Support

### Goal
Implement native, high-performance support for Tree Grids and hierarchical data structures within the Neo.mjs framework. This Epic focuses on the core "Hierarchical Rows" pattern (a flat virtualized view of tree nodes) ensuring O(1) rendering performance and strict adherence to the Row Pooling architecture.

### Scope & Architecture
The Tree Grid implementation is designed to integrate seamlessly with the existing `Neo.grid.Body` virtual scroller without requiring variable-height rows.

1.  **Data Abstraction (`Neo.data.TreeStore` & `TreeModel`):** 
    The complexity of the hierarchy is entirely managed by the data layer. The `TreeStore` parses hierarchical data (using `parentId` relations) into an internal O(1) `#childrenMap`. Crucially, it exposes a dynamically updated, **flattened 1D array of only the *visible* nodes** to the grid. When a node is expanded, its children are injected into this flat array.
2.  **Visual Representation (`Neo.grid.column.Tree`):** 
    A specialized, nested column component handles the visual state. It utilizes CSS variables (`--tree-depth`) for dynamic indentation math and CSS classes for toggle icon states, ensuring rapid recycling during scroll events without DOM manipulation.
3.  **Accessibility (WAI-ARIA):** 
    By leveraging the flattened array approach, the virtual scroller naturally provides gapless, contiguous `aria-rowindex` values. The implementation is augmented with `aria-expanded`, `aria-level`, `aria-setsize`, and `aria-posinset` attributes mapped directly from the records (utilizing "Soft Hydration" where necessary).
4.  **Interaction:** 
    Expansion and collapse interactions are handled via event delegation on the Grid Body, triggering state mutations in the Store, which subsequently fire events (`expand`, `collapse`) for View Controller consumption.

*Note: Variable-height "Row Expanders" (nested content bodies) are outside the scope of this Epic and would require a separate architectural enhancement to the ScrollManager.*

## Timeline

- 2026-03-09T15:45:19Z @tobiu cross-referenced by #9418
- 2026-03-09T15:46:10Z @tobiu cross-referenced by #9419
- 2026-03-09T18:16:54Z @tobiu cross-referenced by #9422
### @tobiu - 2026-03-10T13:35:16Z

https://github.com/user-attachments/assets/4787104c-fb87-479f-8c9a-00abc29c1ca7

wip

- 2026-03-12T14:20:26Z @tobiu cross-referenced by #9449
- 2026-03-17T18:48:48Z @tobiu cross-referenced by PR #9501
- 2026-06-08T00:45:43Z @neo-opus-ada cross-referenced by #9409
### @github-actions - 2026-06-09T05:37:11Z

This issue is stale because it has been open for 90 days with no activity.

- 2026-09-15T08:38:12Z @neo-opus-grace cross-referenced by #15000
- 2026-09-16T08:26:59Z @neo-opus-grace cross-referenced by #18762
- 2026-10-12T14:03:20Z @neo-gpt-euclid cross-referenced by #19658
- 2026-10-28T11:48:26Z @neo-opus-vega cross-referenced by #20166
- 2026-11-03T05:04:02Z @neo-opus-grace cross-referenced by #20212
### @neo-gemini-maryam - 2026-11-05T10:41:29Z

### Backlog Premise-Read & Epic Resolution Check (`neomjs/neo#15000`, `2026-11-05`)

- **Current Code Reality:**
  - **All `33/33` linked sub-issues (`#9405`–`#9448`) are `CLOSED`.**
  - All 4 architectural pillars in the Epic description are shipped and verified on `dev`:
    1. **Data Abstraction:** `src/data/TreeStore.mjs` & `src/data/TreeModel.mjs` (`#childrenMap`, `#allRecordsMap`, flat 1D visible projection, `expand()`, `collapse()`, `expandAll()`, `collapseAll()`, path materialization, Turbo Mode, and hierarchical `doSort` / `filter`).
    2. **Visual Representation:** `src/grid/column/Tree.mjs` and `src/grid/column/component/Tree.mjs` (`--tree-depth`, toggle icon states, cell pooling).
    3. **Accessibility (WAI-ARIA):** `aria-expanded`, `aria-level`, `aria-setsize`, and `aria-posinset` managed on `TreeModel` / `TreeStore` and rendered in `src/grid/column/component/Tree.mjs` / `src/grid/Row.mjs`.
    4. **Interaction, Examples & Tests:** `src/grid/Body.mjs`, `examples/grid/tree/`, `examples/grid/treeBigData/`, `test/playwright/unit/data/TreeStore.spec.mjs`, `test/playwright/e2e/grid/Tree.spec.mjs`, and `test/playwright/e2e/grid/TreeBigData.spec.mjs`.
- **Verdict (`op` — for `@tobiu`):** **Recommend Close as Completed (`33/33` sub-issues closed; all Epic deliverables shipped on `dev`).**

— Maryam (Gemini 4 Argon, Antigravity) · session 70151411-91ec-4496-b8f0-ea3b31ddb60b 🧭


