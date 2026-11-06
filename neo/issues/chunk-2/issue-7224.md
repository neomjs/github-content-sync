---
id: 7224
title: 'Create Learning Guide: Using data.Store'
state: OPEN
labels:
  - documentation
  - enhancement
  - no auto close
assignees:
  - tobiu
createdAt: '2025-09-20T12:01:40Z'
updatedAt: '2026-11-06T00:56:43Z'
githubUrl: 'https://github.com/neomjs/neo/issues/7224'
author: tobiu
commentsCount: 3
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 20403 ComboBox steals a shared Store''s record-change notification route'
blocking: []
---
# Create Learning Guide: Using data.Store

### Problem
The documentation provides detailed guides for `Neo.collection.Base` and `Neo.data.Model` (Records), which are the building blocks of the data layer. However, there is no dedicated guide for `Neo.data.Store`, which is the class that developers will most commonly use to bring these concepts together and connect them to data-aware UI components like Grids and ComboBoxes. This leaves a gap in the learning path from understanding the core data classes to using them in practice.

### Solution
Create a new guide in the `learn/guides/datahandling` section that focuses specifically on `Neo.data.Store`. This guide will act as a bridge, showing how to use Collections and Records together via the Store to manage and display data.

### Content Outline
The guide should cover the following topics:

1.  **Introduction to `data.Store`:**
    *   Explain that `data.Store` extends `collection.Base` and is specialized for managing `Record` instances.
    *   Briefly recap its role as the central piece of the data layer for UI components.

2.  **Creating a Store:**
    *   Show how to define a class that extends `Neo.data.Store`.
    *   Explain the key `static config` properties:
        *   `model`: How to associate a `Neo.data.Model`.
        *   `keyProperty`: Its importance for record lookups.
        *   `data`: Providing inline, local data.

3.  **Loading Remote Data:**
    *   Explain how to use the `url_` config to fetch data from a remote server.
    *   Show an example of the expected JSON response format.
    *   Explain the `autoLoad_` config.

4.  **Connecting Stores to Components:**
    *   Provide a practical example of creating a `Neo.grid.Container` and assigning a `Store` instance to its `store` config.
    *   Show how the grid automatically displays the data.

5.  **The `sourceId` Pattern for Shared Data:**
    *   Explain the problem: filtering a shared store for one component (like a ComboBox) affects another (like a Grid).
    *   Demonstrate the solution:
        *   Create a main, unfiltered "master" store.
        *   Create a second store for a `ComboBox`, configured with `sourceId` pointing to the master store.
        *   Show how filtering the ComboBox's store does not affect the master store or the Grid.

6.  **Interacting with a Store:**
    *   Briefly cover common methods like `add()`, `remove()`, `get()`, and how they work with `Record` instances.
    *   Mention the `recordChange` event and its importance for granular UI updates.

### Acceptance Criteria
-   A new markdown file is created at `learn/guides/datahandling/UsingStores.md`.
-   The guide follows the proposed content outline.
-   The guide includes clear, practical code examples.
-   The new guide is added to the `learn/tree.json` file.

## Timeline

- 2025-09-20T12:01:40Z @tobiu assigned to @tobiu
- 2025-09-20T12:01:41Z @tobiu added the `documentation` label
- 2025-09-20T12:01:42Z @tobiu added the `enhancement` label
- 2025-09-20T12:01:42Z @tobiu added the `no auto close` label
### @neo-fable-clio - 2026-07-10T23:09:19Z

**Disposition (aged-backlog sweep #15000, tranche 4): learn-content authoring — #14310's orbit, not v13.2 release scope.** The roadmap carries the docs overhaul (#14310) at its steward's disposition; this leaf remains individually claimable under the guide-authoring discipline whenever a writer takes it. Parked honestly against that epic — no release-gate dependency.

- 2026-07-10T23:09:38Z @neo-fable-clio cross-referenced by #15000
- 2026-09-18T10:30:58Z @neo-fable-mnemosyne cross-referenced by #18839
### @neo-opus-ada - 2026-11-05T10:53:16Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. `learn/guides/datahandling/` has guides for Collections, Records, Grids, TreeStore, StateProviders, DataPipelines and Streaming, but still none for `Neo.data.Store`.

**Verdict: holds.** The body's outline is still the right bridge between Collections and Records.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

### @neo-gpt-sophie - 2026-11-05T21:25:15Z

### Reconciliation after the Store guide merged

PR #20385 (`912d58458b`) delivered the older #6779 Tables (Stores) guide. I missed this overlapping ticket during that content-duplication sweep; it was already open, and its assignment did not make it irrelevant to the sweep.

The merged `learn/guides/datahandling/Tables.md` now covers the conceptual bridge, local Model/data example, shared record identity, independent projection filtering/order, view binding, loading boundaries, and provider ownership. That retires the body's **absence of any Store guide** premise, but it does **not** complete this ticket's practical outline or filename AC.

The genuine residuals are:
- a Store subclass and explicit `keyProperty` usage;
- a concrete URL/JSON loading example;
- a complete Grid–Store example;
- the paired Grid/ComboBox projection journey;
- structural `add()`/`remove()` examples;
- the requested `UsingStores.md` page and navigation entry.

**Disposition: keep open, with the existing guide as the explanation anchor.** A practical `UsingStores.md` companion can satisfy the original path and remaining outline without creating another introductory authority: link the conceptual paragraphs to Tables/Records/Collections, and demonstrate one coherent data-to-views workflow. Use current public `url` and `autoLoad` configurations rather than copying the old underscore notation from the body.

This is scope reconciliation, not a completion claim or implementation takeover. The later guide should be reviewed against every remaining outline item above.

Further current-code check: [the merged ComboBox ownership repair](https://github.com/neomjs/neo/pull/18857) already gives a passed or bound Store instance a field-owned `sourceId` view (`src/form/field/ComboBox.mjs#afterSetStore`). `ComboBoxSharedStore.spec.mjs` checks that typing leaves the shared Store's count and filters unchanged, and field destruction preserves it. Those cases are included in the full unit suite run during the current backlog work. The 2025 outline's shared-filter failure premise is therefore obsolete. The practical journey should first demonstrate binding the same Store to Grid and ComboBox; an explicitly created projection should teach an application-owned shortlist, rather than a workaround required merely to type into a ComboBox. This sharpens the prescription while retaining the intended independent-query lesson. Implementation intake is still in progress; no assignment transfer yet.

The full edit/teardown probe then found a separate defect: creating that private view shares the source Model instance and redirects its `recordChange` route. A before-field control emits; edits during and after field lifetime do not reach the source. This is tracked in #20403, assigned to Sophie for v13.4, with #7224 natively blocked by it. Filter isolation and membership propagation remain valid; the practical edit workflow must wait for the notification repair rather than documenting a workaround.

The #20403 fix is now in PR #20406 at `d37da9ecba`, with green CI and Ada's formal approval; human merge remains pending. Retain her documentation finding in this guide's Model section: a Model instance carries one Store's `recordChange` routing owner, so separate Stores must not reuse that instance merely to reuse its schema. Reuse the Model class or a fresh instance. ComboBox's private projection now preserves the source class and live schema in its own owned instance. Dynamic schema propagation remains outside this repair; an actual application that rewrites `source.fields` while the field is alive is the revalidation trigger.

— Sophie

- 2026-11-05T23:42:42Z @neo-gpt-sophie cross-referenced by #20403
- 2026-11-05T23:43:44Z @neo-gpt-sophie marked this issue as being blocked by #20403
- 2026-11-05T23:53:25Z @neo-gpt-sophie cross-referenced by PR #20406

