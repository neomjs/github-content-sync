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
updatedAt: '2026-10-12T00:47:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/417'
author: neo-opus-vega
commentsCount: 6
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 282 Port the shared core-corpus scan to repository profiles'
blocking:
  - '[ ] 419 Retire replaced legacy core rows by profile receipt'
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
| source-code rows under `neo-shared/neo` | #282's repository profiles | untouched; preservation control refuses on a count delta | refuse before any delete; once deleting has begun, Chroma has no rollback, so an interruption or a count delta exits non-zero with the partial receipt, and the pre-run backup restores *(aligned 2026-10-12 with Brain #999's review)* | same | AC-3 |
| `ask_knowledge_base` results | #411 AC-4 | no frozen 2026-08-26 conversation row beside a fresh corpus row after this runs | — | — | AC-4 |

## Decision Record impact

`depends-on` #282 (native `blocked_by`). `aligned-with` D#17846 §8.6 (ownership tuple), ADR 0014 §5.2 (`kbSync` and `tenant-repo-sync` remain distinct authorities). Nothing amended.

## Acceptance Criteria

- [ ] **AC-1** — The operation runs only after #282's additive core-profile writer is deployed on the plane beside the fresh corpus manifest (#411) — *deployed code*, not a running `kbSync` scheduler: the legacy sync stays disabled through this run (#253/#411), because a legacy sweep between the two would `delete-upfront` the code rows this ticket must preserve. Before that it refuses with a named code, and no automated lane invokes it.
- ~~**AC-2** — One run on the plane retires every `neo-shared/neo` row whose `type` is `ticket`, `pull` or `discussion`, and its receipt (counts by type, corpus tenant manifest revision) is recorded here, on #411 and on #64.~~ *Moved to Post-Merge Validation 2026-10-11: the run uses the merged tool.*
- [ ] **AC-3** — Preservation control: the count of non-conversation `neo-shared/neo` rows is identical before and after, asserted inside the run; a fixture arm where a source-code row would be caught by the filter makes the run refuse without deleting.
- ~~**AC-4** — After the run, `ask_knowledge_base` cites no conversation row with `repoSlug: 'neo'`; the fresh corpus rows answer instead (#411 AC-4's coexistence boundary closes).~~ *Moved to Post-Merge Validation 2026-10-11, with AC-2.*

## Post-Merge Validation

*(Moved from AC-2 and AC-4 on 2026-10-11 by the author. The implementation PR delivers the tool, AC-1 and AC-3. The run needs the merged tool on the plane, and its owner is this ticket's assignee.)*

- [ ] A fresh backup reads `restorable: true` immediately before the run.
- [ ] One run on the plane, a dry run and then `--apply`, retires every `neo-shared/neo` row whose `type` is `ticket`, `pull` or `discussion`. Its receipt (counts by type, preserved before and after, the corpus tenant's manifest revision) is recorded here, on #411 and on #64.
- [ ] After the run, `ask_knowledge_base` cites no conversation row with `repoSlug: 'neo'`, and the fresh corpus rows answer instead (#411 AC-4's coexistence boundary closes).

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
- 2026-09-23T01:34:40Z @neo-gpt cross-referenced by #419
- 2026-09-23T01:34:51Z @neo-gpt marked this issue as blocking #419
- 2026-09-23T01:41:10Z @neo-opus-vega cross-referenced by #420
### @neo-opus-vega - 2026-09-23T02:29:37Z

**Session handover (sunset 2026-09-23 ~02:30Z) — owner @neo-opus-vega.**

The body is current: AC-1 was reworded at 01:10Z to @neo-gpt's ordering — #282's additive core-profile writer *deployed* beside the fresh #411 manifest, legacy `kbSync` disabled through the run — and their `embedViaShadowSwap` defect-note is an Avoided Trap.

**Gate:** #282 (owner @neo-gpt). #419 (@neo-gpt) owns the old CODE-row retirement and is blocked by this ticket and #282; the two never touch the same rows by construction (this ticket's preservation control refuses any non-conversation delete).

**Pickup:** once #282 lands and #411's manifest exists, implement The Fix as written — scoped delete by `{repoSlug: 'neo', type ∈ ticket | pull | discussion}` with the before/after non-conversation row count asserted inside the run. Nothing started; no branch.

— Vega (Fable 5.1, Claude Code) 🌿

- 2026-09-23T10:20:02Z @neo-opus-vega cross-referenced by PR #423
- 2026-09-23T10:55:20Z @neo-opus-vega cross-referenced by PR #424
- 2026-09-23T11:24:04Z @neo-opus-vega cross-referenced by #64
- 2026-09-23T12:32:50Z @neo-opus-vega cross-referenced by #429
- 2026-09-23T12:33:27Z @neo-opus-vega cross-referenced by #430
- 2026-09-23T12:52:42Z @neo-opus-vega cross-referenced by #432
- 2026-09-23T13:31:04Z @neo-opus-vega cross-referenced by #434
- 2026-09-23T14:17:15Z @neo-opus-vega cross-referenced by #438
- 2026-09-23T14:39:40Z @neo-opus-vega cross-referenced by #440
- 2026-09-24T14:10:17Z @neo-opus-grace cross-referenced by #459
- 2026-09-25T12:42:05Z @neo-opus-vega cross-referenced by #486
- 2026-09-28T09:15:36Z @neo-gpt-emmy cross-referenced by #17416
- 2026-09-28T09:34:16Z @neo-gpt-emmy cross-referenced by #597
- 2026-09-28T11:06:25Z @neo-gpt-emmy cross-referenced by PR #602
### @neo-opus-vega - 2026-10-10T15:17:21Z

**Gate state, 2026-10-10 (Vega, owner):** AC-1's precondition is met on the served plane — #282 (core profiles) closed 2026-09-23 and #402 (the corpus tenant) 2026-09-22, both in Brain `93079328` served since 15:07Z today; the plane env sets no `NEO_ORCHESTRATOR_KB_SYNC_*` key, so the legacy sync runs only if its default enables it (to be read from `ai/configBase.mjs:2217` at implementation, never assumed).

One precondition the body does not name, added here: **the run waits for a verified Knowledge Base backup.** The Memory Core healthcheck's backup axis reads `degraded` (`off-host-durability-unmet`, `backup-retry-exhausted`, `backup-state-conflict`) today, and a scoped delete over `neo-shared/neo` conversation rows is the one operation on this list that a wrong filter cannot undo without a restore point. Falsifier for the defer: a backup receipt naming the KB collection at a revision ≥ `93079328`. Until then this stays the next cleanup-sized Brain lane I hold, after the v13.2 window.

— Vega (Claude Fable 5.1, Claude Code) 🌿


- 2026-10-10T16:27:06Z @neo-opus-vega cross-referenced by #968
- 2026-10-10T16:43:08Z @neo-opus-vega cross-referenced by #974
- 2026-10-10T17:08:29Z @neo-gpt cross-referenced by PR #971
### @neo-opus-vega - 2026-10-11T15:57:16Z

**Gate state, 2026-10-11 (Vega, owner):** the backup precondition from 10-10 is met. Today's bundle `backup-2026-10-11T13-16-01.201Z` was written at Brain `7ca66028` (a descendant of `93079328`); its receipt reads `success`, `restorable: true`, with the Knowledge Base at 121,307 rows and integrity `pass` (source count = bundle count). The full reading is on #969 ([receipt 6110856083](https://github.com/neomjs/neo-agent-brain/issues/969#issuecomment-6110856083)).

Nothing gates the run now except the timing I set on 10-10: it stays after the v13.2 window, because the cut is today and this is a delete on the shared plane's Knowledge Base. Next step at pickup: read the legacy sync default (`ai/configBase.mjs`), then implement The Fix with the preservation control.

— Vega (Opus 5.5, Claude Code) 🌿


### @neo-opus-vega - 2026-10-11T23:30:05Z

**Gate state, 2026-10-11 (Vega, owner): clear to implement, with one correction to my 10-10 note.**

- **Legacy sync.** My 10-10 comment said the plane env sets no `NEO_ORCHESTRATOR_KB_SYNC_*` key. It does: the orchestrator runs with `NEO_ORCHESTRATOR_KB_SYNC_ENABLED=false`, from the #253 cut's compose fragment. It is not in the repo's compose files. The pin is no longer what keeps code rows safe, though. At the served `7ca66028`, `kbSync` runs #282's `embedCoreCorpusProfiles`, which writes both repository profiles additively under their own stamps and "never treats a missing row in the new profiles as authority to delete an old one" (`DatabaseService.mjs`). The `delete-upfront` sweep that AC-1 guards against is gone by construction. The pin is a second safeguard.
- **#282, #402, #411:** closed, all in `7ca66028`.
- **Backup:** restorable at 13:16Z today (#969, receipt 6110856083). The run takes a fresh backup first and holds the heavy-maintenance lease, as `syncKnowledgeBase.mjs` does, so no KB writer runs alongside the preservation count.

Next: the implementation PR. It is a script in `ai/scripts/maintenance/` beside its siblings, plus a unit spec with the refusal fixture. The plane run follows review, merge and that backup.

— Vega (Opus 5.5, Claude Code) 🌿


- 2026-10-11T23:44:26Z @neo-opus-vega cross-referenced by PR #999
- 2026-10-12T00:12:49Z @neo-opus-vega referenced in commit `bd011b3` - "feat(kb): bind the retirement's replacement to the configured corpus and keep its receipt on interruption (#417)

The replacement check now resolves the configured tenant repository that is
the corpus source through the ingestion authority and requires its accepted
manifest, from the configured extraction identity, plus conversation rows of
its own; another repository's rows no longer admit a run. A failing delete
page or recount now throws with the receipt of confirmed pages, and the CLI
prints it through failureReport()."
### @neo-opus-vega - 2026-10-12T00:43:46Z

**Session handover (sunset 2026-10-12 ~00:45Z), owner @neo-opus-vega.**

**State.** Brain #999 is at `bd011b39`, with CI green (23 pass, 1 skipped). It waits on @neo-gpt's Round 2: the seat is re-requested and woken, over review 5485608573 and response 6115545115. Both RAs are addressed:
- the replacement is bound to the configured corpus through `listConfiguredTenantRepos()` + `getTenantManifest()`, with six named zero-delete refusals;
- interruptions throw `KB_LEGACY_RETIRE_INTERRUPTED` with the partial receipt.

The plane dry run of that head at 00:31Z is clean: 38,660 rows selected, 29,320 preserved, no refusals, replacement `github-content-sync` at extraction identity `9785ca5a…`. The receipt is in #999's Test Evidence.

**Pickup.**
1. **Round 2:** if it approves, the merge is @tobiu's. If it comments, answer in a new response comment.
2. **After merge, Post-Merge Validation** (owner #64, see the body):
   - a fresh backup with `restorable: true`;
   - in the orchestrator container, a dry run then `--apply --confirm CONFIRM_RETIRE_LEGACY_CONVERSATION_ROWS`. This needs the merged file on the plane, either through a recreate at a revision that contains the merge, or through the same stdin route the dry runs used (relative imports rewritten to `/app/ai/`);
   - record the receipt here, on #411 and on #64;
   - check that `ask_knowledge_base` cites no `repoSlug: 'neo'` conversation row.
3. Expect the run to queue behind the heavy-maintenance lease. `summary` and `tenant-repo-sync` both held it tonight, and the CLI refuses with `KB_LEGACY_RETIRE_LEASE_HELD` without reading anything.

**Related, not this ticket's:** #282's profile writer has never run on the plane, because `KB_SYNC_ENABLED=false` is pinned by the #253 cut fragment. A defect-note went to the swarm, and @neo-gpt was DM'd for #419.

— Vega (Opus 5.5, Claude Code) 🌿


### @neo-gpt - 2026-10-12T00:47:10Z

## Session handover — scope: convergent

The operator explicitly ended this session at 02:40 Europe/Berlin. This is a reviewer handover, not approval of a plane run. Vega retains implementation/execution ownership; @neo-gpt retains the sole requested review seat.

[PR #999](https://github.com/neomjs/neo-agent-brain/pull/999) is OPEN/CLEAN at `bd011b392ff2bb157a8ea7e7e96f04faff3b181a`, still CHANGES_REQUESTED pending my Round 2. My [Round-1 review 5485608573](https://github.com/neomjs/neo-agent-brain/pull/999#pullrequestreview-5485608573) contains exactly two actions: actual replacement-corpus/manifest admission and partial mutation-failure receipts. Both failures were executed against isolated collections at `a701d55e`; no plane deletion ran.

The author reports both addressed in [response 6115545115](https://github.com/neomjs/neo-agent-brain/pull/999#issuecomment-6115545115). A later exact-head dry run is now in the PR body: configured `github-content-sync` manifest identity, no refusals, 38,660 selected / 29,320 preserved. These are author receipts for the next reviewer to verify, not a completed review verdict.

**First fresh-session pickup:** context recovery, live head/seat/CI, then disposition-only Round 2 over the two original actions verbatim. The ordinary GPT Request Changes round is already spent. Do not open a new findings list or run apply during review. Actual execution still requires the merged tool, fresh restorable backup and the existing #64/#411 receipts.

The separate #419 code-row retirement remains blocked on actual new-profile manifests/landed replacements and this predecessor; the reported zero Brain-profile rows are a lead to measure, not deletion authority. Do not flip the KB scheduler or sweep code/Skill/ReleaseNote/concept rows.

Both dedicated checkouts are being refreshed on dev; no uncommitted source, tests or running Root jobs remain. Durable fresh-session self-DM and the final Sandman memory will carry exact local pins and wake-route state.

Origin Session ID: 2d8feac7-c60d-4883-8059-37b6e148768b


