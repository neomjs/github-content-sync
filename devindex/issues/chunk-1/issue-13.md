---
id: 13
title: Viewport size swap rewrites the whole cls array to change one class
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-09-15T11:32:33Z'
updatedAt: '2026-09-15T13:20:13Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/13'
author: neo-opus-vega
commentsCount: 1
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
closedAt: '2026-09-15T13:20:13Z'
---
# Viewport size swap rewrites the whole cls array to change one class

## Context

`DevIndex.view.Viewport#afterSetSize` reads the component's entire `cls` array, edits it, and assigns the whole value back in order to change **one** class — the `devindex-size-*` marker.

This file is named by `neomjs/neo#15202`, which inventories six application sites doing exactly this. That ticket was written before DevIndex moved out of the engine repository, so it still lists the file at `apps/devindex/view/Viewport.mjs` **in `neomjs/neo`**, where it no longer exists. The work is real; only the address in that ticket is stale. Its AC-4 and AC-9 cannot be discharged from the engine repo, which is why this leaf exists here.

**Provenance worth keeping.** A Memory Core sweep on the symptom surfaced session `c2f788f3` (2026-02-23): this `afterSetSize` / `monitorSize` / `size_` block was added to DevIndex under ticket `#9273` **specifically to mirror `apps/portal/view/Viewport.mjs`**, whose mobile size tracking DevIndex was missing. Portal's half has now been migrated in `neomjs/neo#18735`. So this change is not a divergence from Portal — it is what keeps the two in parity.

Live latest-open sweep: checked all 3 open issues at 2026-09-15T11:30:46Z (`#1`, `#9`, `#10`); no equivalent. A2A in-flight claim sweep over the latest 30 messages: no competing claim on this surface. Memory Core rationale sweep: `#9273` is the only prior decision, and it supports this change rather than opposing it. Own-assignment sweep: no open assignments in this repository.

## The Problem

```js
afterSetSize(value, oldValue) {
    if (value) {
        let me  = this,
            cls = me.cls;

        NeoArray.remove(cls, 'devindex-size-' + oldValue);
        NeoArray.add(   cls, 'devindex-size-' + value);
        me.cls = cls;
```

The transition already knows both class names. Reading the aggregate to express it couples this application to every **other** class on the component — the framework's `baseCls`, the layout's, and any a caller added — and to the fact that `cls` is an array today. Neither is this application's concern, and neither is a contract the engine promises to keep.

## The Architectural Reality

`cls` is a reactive config owned by `Neo.component.Base`. Its getter is not a window onto internal state:

```js
beforeGetCls(value) { return value ? [...value] : [] }
```

Every read returns a **fresh copy**, which is why the current code is correct rather than lucky — it mutates a detached snapshot. The engine exposes `addCls()`, `removeCls()` and `toggleCls()`, each of which performs that same copy-mutate-assign internally. Using them is not a behaviour change; it is the same operation named at the boundary that owns it.

`NeoArray` is imported at line 3 solely for these two calls, so it becomes unused.

## The Fix

In `apps/devindex/view/Viewport.mjs`:

- `afterSetSize` calls `me.removeCls('devindex-size-' + oldValue)` and `me.addCls('devindex-size-' + value)`.
- The local `cls` binding and the `me.cls = cls` write-back are removed; `let me = this` stays, since `me.stateProvider.setData({size: value})` follows.
- The now-unused `NeoArray` import is removed.

No engine, config-shape or state-provider change. `me.stateProvider.setData({size: value})` is untouched.

## Decision Record impact

`none`. This is an application-layer consumer change using an existing public component API.

## Acceptance Criteria

- [ ] `afterSetSize` no longer reads the aggregate `cls` value and no longer assigns a transformed aggregate back.
- [ ] The size transition removes only the prior `devindex-size-*` class and adds the next one, preserving every unrelated class.
- [ ] The `NeoArray` import is removed, and no other reference to it remains in the file.
- [ ] `me.stateProvider.setData({size: value})` still runs on every size change.
- [ ] Repeated transitions stay duplicate-free and idempotent.
- [ ] A scoped source check finds no aggregate read-transform-reassign pattern left in the file.
- [ ] Post-merge: the responsive layout still switches correctly across breakpoints, including the `x-small` hamburger branch that `#9273` added this block for.

## Out of Scope

- Any other `cls` assignment that replaces a complete authored value without reading the current aggregate.
- `ViewportController.mjs`, `ViewportStateProvider.mjs`, and `HeaderToolbar.mjs`.
- The `monitorSize` / `getSize` / `onResize` mechanism itself.
- Engine changes of any kind, including widening the `addCls` / `removeCls` JSDoc, which documents `{String}` while `NeoArray` accepts arrays.

## Avoided Traps

**Passing an array to `removeCls()`.** `NeoArray.remove` accepts "an item or Array of items", so `removeCls(['a','b'])` works at runtime — but the component method documents `@param {String}`. Relying on it would be depending on undocumented behaviour, and widening that JSDoc is an engine change. Single-value calls only.

**Assuming the extra write is free.** The aggregate form issues one `cls` write; this form issues two, and `afterSetCls` ends in `update()`. That difference was measured rather than assumed in `neomjs/neo#18735`: a `MutationObserver` on the element's `class` attribute recorded identical DOM class sequences for both forms in the same worker tick, against a positive control that does produce an intermediate frame when the writes land in separate ticks.

## Related

- `neomjs/neo#15202` — the parent inventory; this file is its sixth site, at its pre-split address.
- `neomjs/neo#18735` — the engine-repo companion covering the other five sites, including Portal's identical `afterSetSize`.
- `#9273` — added this block to mirror Portal.

Origin Session ID: 8c22722a-cea7-40d1-9400-f8455c82efe4

Handoff Retrieval Hints: `query_raw_memories("devindex viewport size class aggregate cls migration")` · Retrieval Hint: the engine-side pattern and its measurement live in `neomjs/neo` commits `9e251acbd1..ff75441172`.

## Timeline

- 2026-09-15T11:32:34Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-15T11:32:35Z @neo-opus-vega added the `enhancement` label
- 2026-09-15T11:32:35Z @neo-opus-vega added the `ai` label
- 2026-09-15T11:33:44Z @neo-opus-vega cross-referenced by PR #14
- 2026-09-15T11:34:25Z @neo-opus-vega cross-referenced by #15202
### @neo-opus-vega - 2026-09-15T13:20:13Z

Closing not-planned. The premise is falsified: read-mutate-assign on the aggregate `cls` array is the engine's intended idiom (`component/Base.mjs#afterSetDisabled`, `Icon.mjs:41`, `IconLink.mjs:68`, `CheckBox.mjs:227`), made safe by `beforeGetCls` returning a copy. `removeCls()` + `addCls()` costs a second vdom cycle because `afterSetCls` ends in `update()` and the in-flight guard defers rather than merges. The original code was correct; nothing to fix. Detail on PR #14. — Vega 🌿

- 2026-09-15T13:20:14Z @neo-opus-vega closed this issue

