---
id: 417
title: Retire the legacy neo-owned conversation rows once the core profiles are live
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-22T23:39:57Z'
updatedAt: '2026-09-23T01:10:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/417'
author: neo-opus-vega
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 282 Port the shared core-corpus scan to repository profiles'
blocking: []
---
# Retire the legacy neo-owned conversation rows once the core profiles are live

## Context

#402 retires the three Knowledge Base Sources that stamped GitHub conversations under `neo-shared/neo`; the corpus now arrives as its own tenant (`github-content-sync`, #411). The rows the old Sources wrote — the conversation corpus frozen at 2026-08-26 — stay in the store until something retires them. #282 owns the plan for the full re-embed / legacy-row migration of the `neo-shared/neo` stamp and **explicitly does not execute it** (its AC: *"evidenced before enabling the new core profiles, but is not executed by the implementation PR"*; out of scope: *"running a re-embed, deleting legacy rows"*). @neo-gpt's #412 review made the gap concrete: #411 cannot own the retirement either, because the only lane that could perform it today — the legacy `kbSync` — scans a hierarchy #282 records as red and would take source-code rows with the conversations. This ticket is the executing owner.

## The Problem

`VectorService.embed()`'s default stale strategy (`delete-upfront`, `VectorService.mjs:506`) is scoped to the whole `neo-shared/neo` tuple: everything an earlier sync stamped there and the current run does not re-emit is deleted. Run after #402 by the legacy path, that is correct for the conversation rows (nothing emits them any more) and catastrophic for source-code rows if the scan itself is broken — which is #282's finding (`node_modules/neo.mjs/apps` 92.9 % < 93 % floor, `ai` 0/174). So the retirement of ~thousands of frozen conversation rows must be its own scoped operation, run only after #282 has replaced the legacy scan with repository-bound core profiles, and it must prove it touched no source-code row.

## The Architectural Reality

- Legacy stamp: `{tenantId: 'neo-shared', repoSlug: 'neo'}` on both source-code rows (`ApiSource` and siblings) and the frozen conversation rows (`type` ∈ `ticket` | `pull` | `discussion`, `sourcePath` under `resources/content/…`).
- Corpus rows: `{tenantId: 'neo-shared', repoSlug: 'github-content-sync'}`, `type` the same three, `customMeta.origin` = the conversation's origin. Reconciliation is per `repoSlug` (`kbReconciliationEngine.mjs` `diffTenantManifest`), so the two populations never see each other as stale — the retirement has to be explicit.
- Manifests: `IngestionService.getTenantManifest({tenantId, repoSlug})` exposes `pathsAfterPush` per repo; a scoped delete by `{repoSlug: 'neo', type ∈ conversation types}` (or by `sourcePath` prefix) is the operation, with the row census before/after as the receipt.

## The Fix

1. A maintenance operation (script or `manage_knowledge_base` action — decided at implementation against the existing maintenance surface, no new daemon) that deletes rows where `repoSlug === 'neo'` and `type ∈ {ticket, pull, discussion}` (equivalently `sourcePath` under the legacy `resources/content/` roots), and nothing else.
2. A preservation control run in the same invocation: the count of `neo-shared/neo` rows whose `type` is NOT a conversation type before and after — must be equal, or the run refuses to commit.
3. A receipt: rows retired by type, source-code rows before/after, the corpus tenant's manifest revision at the time — recorded on #411 and #64.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| legacy conversation rows under `neo-shared/neo` | #402 (Sources retired) + #282 (core profiles) | deleted by explicit scoped operation, only after #282 | none — nothing deletes them implicitly; the legacy sync stays off | KB handbook maintenance section | AC-2 receipt |
| source-code rows under `neo-shared/neo` | #282's repository profiles | untouched; preservation control refuses on a count delta | refuse, no partial delete | same | AC-3 |
| `ask_knowledge_base` results | #411 AC-4 | no frozen 2026-08-26 conversation row beside a fresh corpus row after this runs | — | — | AC-4 |

## Decision Record impact

`depends-on` #282 (native `blocked_by`). `aligned-with` D#17846 §8.6 (ownership tuple), ADR 0014 §5.2 (`kbSync` and `tenant-repo-sync` remain distinct authorities). Nothing amended.

## Acceptance Criteria

- [ ] **AC-1** — The operation runs only after #282's additive core-profile writer is deployed on the plane beside the fresh corpus manifest (#411) — *deployed code*, not a running `kbSync` scheduler: the legacy sync stays disabled through this run (#253/#411), because a legacy sweep between the two would `delete-upfront` the code rows this ticket must preserve. Before that it refuses with a named code, and no automated lane invokes it.
- [ ] **AC-2** — One run on the plane retires every `neo-shared/neo` row whose `type` is `ticket`, `pull` or `discussion`, and its receipt (counts by type, corpus tenant manifest revision) is recorded here, on #411 and on #64.
- [ ] **AC-3** — Preservation control: the count of non-conversation `neo-shared/neo` rows is identical before and after, asserted inside the run; a fixture arm where a source-code row would be caught by the filter makes the run refuse without deleting.
- [ ] **AC-4** — After the run, `ask_knowledge_base` cites no conversation row with `repoSlug: 'neo'`; the fresh corpus rows answer instead (#411 AC-4's coexistence boundary closes).

## Out of Scope

- Porting the core scan (#282) and enabling the legacy sync — this ticket runs after them, never instead of them.
- The corpus tenant's activation (#411) and the extractor (#402).
- Any full re-embed of source-code rows; if #282's plan requires one, that is #282's successor, and this ticket's control forbids touching those rows.
- **The legacy `neo-shared/neo` CODE rows** (added 2026-09-23 from @neo-gpt's #282 intake fork). #282 writes its Engine/Brain profiles additively (`deleteStale: false`) until a migration owner and receipt exist, so the old code rows coexist with the new profile-stamped ones after it lands. Retiring them is #282's AC-9 owner's, never this ticket's: this ticket's preservation control refuses any run that would delete a non-conversation row, by construction. Ordering between #282-additive and this ticket is free; this ticket still waits for #282 only because the legacy `kbSync` must not run in between.

## Avoided Traps

- ⛔ Do not perform this retirement through a legacy `kbSync` run — `delete-upfront` over the whole tuple is exactly the mechanism that would take source-code rows (#282, #412 review).
- ⛔ Do not key the deletion on `sourcePath` alone: the corpus rows carry `neo/issues/…` paths and the legacy rows `resources/content/issues/…`; the `repoSlug` is the discriminator, the type the scope.
- ⛔ Do not perform the retirement through `VectorService.embedViaShadowSwap`: the shadow is built from the current input alone (`VectorService.mjs:2088-2135`) and renamed over the canonical collection (`:2260-2275`), so a one-repo swap drops every other tenant's rows (@neo-gpt's defect-note, 2026-09-23). Scoped deletion, or a full unified-corpus shadow candidate, are the only shapes.

## Related

#402 (the Sources retired) · #411 (activation; its AC-5 names this ticket as the retirement owner) · #282 (the gate) · #64 (receipt) · #253 · D#17846 §8.6 · PR #412 review 5285178042

Live latest-open sweep: latest 20 open Brain issues at 2026-09-22T23:38Z — none equivalent; nearest #282 (plans, does not execute), #411, #402. Search over all states for "re-embed / migration / retire legacy rows": no executing leaf exists. A2A in-flight sweep (mailbox read through 23:38Z): no claim; @neo-gpt's review is the trigger. Memory Core sweep: prior migrations (embedding model 2025-12, userId tagging 2026-05) were whole-store or tag-only, none scoped by tuple + type. Own-assignment sweep: #402, #411, #415, #237, #64, #65, #23 — none equivalent. Structure map: N/A until implementation picks script vs `manage_knowledge_base` action; no new daemon.

Origin Session ID: fc04c361-0cae-4a80-9506-fa2ef4785d2b
Retrieval Hint: "retire legacy neo-owned conversation rows neo-shared/neo scoped delete preservation control after #282 core profiles"


## Timeline

- 2026-09-22T23:39:57Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-22T23:39:59Z @neo-opus-vega added the `enhancement` label
- 2026-09-22T23:39:59Z @neo-opus-vega added the `ai` label
- 2026-09-22T23:39:59Z @neo-opus-vega added the `agent-os` label
- 2026-09-22T23:40:10Z @neo-opus-vega marked this issue as being blocked by #282
- 2026-09-22T23:40:31Z @neo-opus-vega cross-referenced by #411
- 2026-09-22T23:40:32Z @neo-opus-vega cross-referenced by PR #412
- 2026-09-22T23:45:42Z @neo-gpt cross-referenced by #282

