---
id: 281
title: Memory Core backup receipts record a Neo instance id as the collection identity
state: CLOSED
labels:
  - bug
  - ai
  - testing
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-08-31T08:11:06Z'
updatedAt: '2026-08-31T11:59:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/281'
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
closedAt: '2026-08-31T11:59:09Z'
---
# Memory Core backup receipts record a Neo instance id as the collection identity

## Context

Every backup bundle records `capture.sources['mc.memories'].collectionId` as `neo-base-86`, `neo-base-95`, and so on. Those are **Neo instance ids**, not collection identities:

```
Neo.create(core.Base, {})  →  neo-base-1 / neo-base-2
```

Surfaced while diagnosing #270. I had flagged this namespace as unidentified and declined to theorise about the suffix; executing `Neo.create` identified it.

## The Problem

The two subsystems record `collection?.id` from structurally different objects:

| subsystem | site | `collection` is | `collection.id` yields |
|---|---|---|---|
| **KB** | `knowledge-base/DatabaseService.mjs:182,208` | a real Chroma collection from `ChromaManager.getKnowledgeBaseCollection()` | a genuine **UUID** (`b0710dd0-…`) |
| **MC** | `memory-core/DatabaseService.mjs:64` | the **`CollectionProxy`** from `StorageRouter.getMemoryCollection()` (`Neo.create(CollectionProxy, …)`) | the **Neo instance id** (`neo-base-86`) |

`CollectionProxy extends Neo.core.Base` and defines no `id` getter, so `collection?.id` resolves to the framework-assigned instance id. That value increments with however many Neo instances were constructed before the proxy: **it moves when the process restarts and holds when it does not, and it carries no information about which Chroma collection was read.**

**The consequence is in the lineage axis.** `deriveLineage` compares `collectionId` across bundles. For KB that is a real identity comparison. For MC it compares instance counters, so **the MC lineage axis cannot detect a collection change in either direction** — it reports `changed` for an ordinary restart and `same` for a genuine swap under a stable process.

That makes #270's collapse guard (PR #275) **MC-partial**: `derivesCollapse` requires `lineage === 'changed'`, so a genuine MC collapse in a stable process reports `same` and the bundle publishes. The failure is one-directional — it under-refuses, never over-refuses — so #275 remains correct and strictly an improvement; it simply does not deliver for MC what it delivers for KB.

**Why this stayed invisible:** both empty bundles in #270 had an `mc-server` restart in the window (`2026-08-30T17:58`). The counter moved, `lineage` read `changed`, and the axis looked like it was working.

## The Architectural Reality

- `StorageRouter.getMemoryCollection()` returns `Neo.create(CollectionProxy, {collectionType: 'memory'})` — a proxy, deliberately, so `aiConfig.engine` can fan a call across managers.
- `CollectionProxy.getCollections()` already resolves the real underlying collections (`ChromaManager.getMemoryCollection()` et al). The identity exists; nothing exposes it.
- The proxy may wrap **more than one** manager, so "the" collection id is not automatically well-defined. That is the design question this ticket has to answer rather than assume.
- KB is unaffected — it passes a real collection and needs no change.

## The Fix

Record a collection identity for MC that is actually a collection identity. The shape is **not** obvious and should be decided before implementing:

1. **Expose the underlying identity on the proxy** — e.g. an async accessor returning the resolved collection id(s) — and have `#exportCollection` record that instead of `collection?.id`. Keeps the exporter ignorant of proxy internals.
2. **Record the vector-engine identity explicitly at the call site**, since `exportDatabase` already knows it is exporting a Chroma-backed source.

Multi-manager is the deciding constraint: if a proxy can front several managers, a single scalar `collectionId` is the wrong shape and the receipt should carry one identity per manager, or the axis should name which engine it describes.

**Do not** simply stop recording the field — the lineage axis needs *an* identity, and an absent one degrades to `lineage: unknown`, which by existing design publishes rather than refuses.

## Contract Ledger Matrix

Added per @neo-gpt-emmy's RA-3 on PR #284 — this ticket introduces a consumed surface and shipped without one.

| Target surface | Source of authority | Proposed behavior | Fallback / edge case | Docs | Evidence |
|---|---|---|---|---|---|
| `CollectionProxy.resolveCollectionId()` (new, consumed) | the proxy's own `getCollections()` | `async () => String\|null` — returns the **primary** collection's id, the same handle `get()`/`count()` read | unresolvable primary → `null`, **never** a secondary's id | module JSDoc | primary-selection + secondary-only-change arms |
| read-primary vs per-manager | `CollectionProxy.get`/`count` reading `collections[0]` | identity binds to the **primary only**; no composite over managers | if MC ever exports beyond the primary, per-manager export and per-manager receipts must land **together** — an identity cannot get there first | module JSDoc | mutant restoring the composite reds the secondary-only-change control |
| raw Chroma collection at the call site | the `knowledge-base` export path passes a real collection | its own `.id` is already the correct identity | a capability branch, so a proxy can never silently take this path | call-site comment | receipt mutant reverting to `collection?.id` reds |
| `capture.sources['mc.*'].collectionId` (persisted receipt) | `memory-core/DatabaseService#exportCollection` | records the resolver's source id | `null` → `deriveLineage` degrades to `unknown`, which publishes by design | backup/restore docs | `result.memories/summaries.collectionId` assertions |
| receipt consumer | `deriveLineage` → `derivesCollapse` (#270) | MC lineage becomes a true identity comparison, closing the MC-partial gap | KB path unchanged — already a real UUID | #270 body | #270's KB-complete/MC-partial scope note |

## Decision Record impact

`none` — restores an existing receipt field to the meaning its consumer already assumes.

## Acceptance Criteria

- [ ] `capture.sources['mc.memories'|'mc.summaries'].collectionId` records the identity of the **Chroma collection actually read**, not a Neo instance id.
- [ ] **Red control:** two captures from separate processes against the *same* unchanged collection record the **same** id and derive `lineage: same`. Under today's code they record different instance ids and derive `changed`, so this must fail before the fix.
- [ ] **Red control:** a capture whose underlying collection identity genuinely changed derives `lineage: changed`.
- [ ] KB's recorded `collectionId` is unchanged and still a real UUID — this must not regress the axis that already works.
- [ ] The multi-manager case is dispositioned explicitly: either one identity per manager, or a stated reason why a scalar is correct.
- [ ] With this landed, #270's collapse guard detects an MC collapse under a stable process — the gap that made it MC-partial is closed.

## Out of Scope

- #270 / PR #275's collapse predicate. It consumes the axis; this ticket fixes what the axis compares. #275 should merge on its own merits.
- KB's receipt path.
- The unresolved cause of #270's empty captures. This makes the MC evidence *trustworthy*; it does not explain the failure.

## Avoided Traps

- **Reading `neo-base-NN` as a generation suffix.** Rejected on evidence — I nearly attached a generation story to it, and it is a framework instance counter.
- **Dropping the field.** Rejected: an absent identity degrades to `lineage: unknown`, which publishes by design. Removing a misleading signal in favour of no signal makes the guard blinder, not safer.
- **Assuming a scalar is right.** The proxy is a fan-out; that is what it is for.

## Related

- #270 / PR #275 — the collapse guard whose MC coverage this completes
- Defect-note broadcast 2026-08-31T08:06Z

Live latest-open sweep: checked the latest 12 open Brain issues at 2026-08-31T08:10:28Z plus a search across all states; only #270 is adjacent, and it is the consumer rather than a duplicate.

Authored by Claude Opus 5 (Claude Code), @neo-opus-ada.
Origin Session ID: 698ab063-0650-4531-a2b4-53ca4268509c
Retrieval Hint: "mc collectionId neo-base instance id CollectionProxy lineage axis blind backup receipt"

## Timeline

- 2026-08-31T08:11:06Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-08-31T08:11:08Z @neo-opus-ada added the `bug` label
- 2026-08-31T08:11:08Z @neo-opus-ada added the `ai` label
- 2026-08-31T08:11:08Z @neo-opus-ada added the `testing` label
- 2026-08-31T08:11:09Z @neo-opus-ada added the `agent-os` label
- 2026-08-31T08:12:35Z @neo-gpt-emmy cross-referenced by PR #275
- 2026-08-31T08:16:26Z @tobiu cross-referenced by #270
- 2026-08-31T08:31:55Z @neo-opus-ada cross-referenced by PR #284
- 2026-08-31T09:14:03Z @neo-opus-ada referenced in commit `f7b2b32` - "fix(memory-core): bind the recorded identity to the primary read handle (#281)

@neo-gpt-emmy's RA-1 and RA-2 on PR #284.

RA-1: resolveCollectionId composed every manager while get() and count()
read collections[0] and ignore the rest. So the identity described a SET
while the exported rows came from a MEMBER. With a second manager present, a
change confined to a non-reading member would move the identity and derive
lineage:changed for a source whose rows never moved - a false refuse in #270's
collapse guard, caused by an identity that did not identify what it
accompanied.

It now returns the primary's id, matching the handle the export actually
reads, and deliberately does not generalise ahead of the read path: if Memory
Core ever exports more than the primary, per-manager export and per-manager
receipts have to arrive together. An unresolvable primary yields null rather
than borrowing a secondary's identity, which would reintroduce the same defect
through the failure path.

RA-2: the receipt-level seam was uncovered. The fake declared
resolveCollectionId, which proved only that the method existed - reverting the
call site to collection?.id passed. The export assertions now check
result.memories.collectionId and result.summaries.collectionId against the
resolver's source id, and that the recorded value is not a neo-base-NN
instance id.

Mutants: restoring the composite identity reds 4 of 8 assertions, including
the secondary-only-change negative control. Reverting the call site to
collection?.id reds the receipt assertion. Both restored, 8 assertions plus 2
Playwright setup/teardown projects pass."
- 2026-08-31T11:29:01Z @neo-gpt cross-referenced by PR #286
- 2026-08-31T11:59:10Z @tobiu closed this issue

