---
id: 974
title: 'Backup bundles compress their JSONL payloads: 9.7 GB a day is embedding floats stored as text'
state: OPEN
labels:
  - enhancement
  - ai
  - performance
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-10T16:42:46Z'
updatedAt: '2026-10-10T17:38:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/974'
author: neo-opus-vega
commentsCount: 2
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
# Backup bundles compress their JSONL payloads: 9.7 GB a day is embedding floats stored as text

## Context

The operator asked why today's bundle is 9.7 GB when the plane was ~3.5 GB a month ago, then asked for compression ("at this size, gzip might be a very valuable enhancement, especially since our neo corpus will grow more", 2026-10-10). Measured on the local plane's retained bundles (`~/.neo-ai/backups`, 31 daily bundles, 250 GB on disk, retention `maxDays 30` / `keepMinimum 3`):

| bundle | total | kb | mc | graph | KB rows |
|---|---|---|---|---|---|
| 2026-09-10 | 6.1 GB | 3.6 GB | 2.4 GB | 0.18 GB | 68,207 |
| 2026-09-27 | 11.5 GB | 8.8 GB | 2.5 GB | 0.23 GB | 157,690 |
| 2026-09-30 | 9.3 GB | 6.5 GB | 2.5 GB | 0.25 GB | 117,570 |
| 2026-10-10 | 9.7 GB | 6.7 GB | 2.7 GB | 0.28 GB | 120,911 |

Row size is constant (~55 KB per KB chunk, ~55 KB per memory); the growth is row count: the `github-content-sync` corpus became a tenant on 09-24 (#402, #282), #602 removed ~40k superseded source revisions on 09-30, and ~38k legacy neo-owned conversation rows still coexist with the corpus (#417 retires them). The live stores are Chroma 16 GB and the graph SQLite 4 GB; the bundle is their text export.

## The Problem

A row is one JSON line whose `embedding` is 4,096 floats serialized as decimal text — 72% of the bytes (52 KB of 72 KB on today's first KB row). Nothing in the lane compresses: `ai/scripts/maintenance/backup.mjs` writes `.jsonl` and every reader (`restore.mjs`, `restoreReceipts.mjs`, the integrity line reader at `backup.mjs:785`, the retention classifier at `:932` / `:1196` / `:1520`) filters on `.endsWith('.jsonl')` and streams the file through `readline` over `fs.createReadStream`. Measured on 1,854 real KB rows from today's export (node `zlib`, brotli quality 4; gzip-6 on a 200 MB slice gave 2.5× for kb and 2.6× for mc, gzip-1 2.3×):

| encoding of the row | raw KB/row | brotli-4 KB/row | vs today |
|---|---|---|---|
| floats as text (today) | 53.9 | 19.4 | 2.8× |
| float32 binary, base64 | 23.7 | 16.6 | 3.2× (lossless) |
| float16 binary, base64 | 12.7 | 8.5 | 6.3× (lossy) |
| row without its embedding | 1.8 | 0.3 | — |

In-process on the 200 MB slice: brotli-4 2.3 s, gzip-6 6.7 s. Today's export takes 217 s; streaming brotli-4 on ~9.4 GB adds roughly 100 s of CPU on a lane that runs at 13:15Z under the heavy-maintenance lease.

Prior art: Grace, 2026-07-03 (Memory Core `d7db576f`), on the KB release artifact: "the vectors aren't the bloat, their JSON-float TEXT encoding is"; fp16 ~2.6× on that artifact; the operator's constraint that shipped vectors spare adopters a re-embed; the fp16-vs-fp32 recall trade-off left for a Discussion. That artifact is a different path (`uploadKnowledgeBase`), the same encoding.

## The Architectural Reality

- Writer: the per-subsystem exporters in `ai/scripts/maintenance/backup.mjs` stream rows to `<bundle>/<subsystem>/<name>.jsonl`; `bundle-meta.json` records counts and the embedding dimension / model, so a reader can already tell what it holds.
- Readers that must accept a compressed payload: `restore.mjs` (`:578-599`, `:675-677`, `:1264`, `:1304`, `:1443`, `:1502`, `:1567-1574`), `restoreReceipts.mjs` (`:62-93`), `backup.mjs`'s integrity line count (`:785`) and the retention classifier (`restorableFor` is decided from counts and non-empty payloads, `:1122-1254`), `redeployPreflight.mjs` (its `RESTORABLE` probe counts rows), `backupCorruptionTimeline.mjs`, `backupStagingResidueCore.mjs`.
- Node core has `zlib.createBrotliCompress` / `createBrotliDecompress` and `createGzip` / `createGunzip`; no new dependency.

Design authority: none found on the payload encoding (searched `backup.mjs`, `restore.mjs`, the maintenance JSDoc, ADR directory for "compress", "gzip", "brotli"); the format is an implementation detail of the lane, and the lossless lever changes no reader contract.

## The Fix

1. **Streaming compression, lossless (this ticket).** The exporters pipe each payload through `zlib.createBrotliCompress({params: {BROTLI_PARAM_QUALITY: 4}})` to `<name>.jsonl.br`; one helper `openBundlePayload(filePath)` returns a line stream for `.jsonl`, `.jsonl.br` and `.jsonl.gz` by extension, and every reader above uses it; the `endsWith('.jsonl')` filters become a shared `isBundlePayload(name)`; `bundle-meta.json` records `payloadEncoding: 'jsonl+br'` per subsystem. Old bundles stay readable and restorable unchanged. Expected: ~3.5 GB per bundle, ~105 GB for 30 days.
2. **Binary float32 embeddings (optional second step, lossless, +15%).** `embedding` as base64 float32 with `embeddingEncoding: 'f32le-base64'` in the meta; the restore writer decodes before upsert. Taken only if the reader change is as small as the writer change.
3. **float16 (not this ticket).** 6.3× but lossy; whether a restored store may differ from the live vectors by fp16 rounding is the recall trade-off Grace's July finding left open, and a Discussion decides it, with the operator's no-re-embed constraint kept either way.
4. **Retention** (`maxDays`, `keepMinimum`) stays as is; it is a separate operator knob.

## Acceptance Criteria

- [ ] AC-1: a bundle written at the head stores every exported payload (kb, mc, graph) as `.jsonl.br`; `bundle-meta.json` names the encoding per exported subsystem; the integrity counts still equal the source counts. The flat copies (`concepts/`, `trajectories/`, `mailbox/`, `ledgers/`) stay `.jsonl` (intake comment: verbatim copies restored verbatim, under 1% of the bytes).
- [ ] AC-2: `restore.mjs`, `restoreReceipts.mjs`, `redeployPreflight.mjs`, the retention classifier and the corruption-timeline reader read `.jsonl.br` and `.jsonl` alike, with one arm per reader on a two-row fixture in each encoding (red-first on the compressed arm).
- [ ] AC-3 *(plane, post-merge)*: the first daily bundle after a cut past this change measures ≤ 40% of the previous day's bytes for kb and mc, and `redeployPreflight` reports `PROCEED_VERIFIED` against it.
- [ ] AC-4: a restore drill from a compressed bundle into a scratch Chroma reproduces the subsystem counts (the existing restore receipt path), recorded here.

## Out of Scope

- fp16 (Fix 3) and the retention policy.
- The KB release artifact (`uploadKnowledgeBase`), which Grace's July finding covers.

## Avoided Traps

- gzip at level 9 or brotli at high quality on a 9 GB stream: measured gains over quality 4 are small and the CPU cost lands inside the heavy-maintenance lease.
- Compressing only the embedding field as a string inside otherwise-uncompressed JSONL: the row text without its vector is 1.8 KB, so the vector is the only thing worth encoding, and whole-stream compression is simpler for every reader.
- Changing the schema version for a lossless encoding change: readers detect by extension and meta, so old bundles need no migration.

## Related

- #968 / #969 (the lane's exit-code fold), #417 (removes ~38k duplicate rows, ~2 GB of today's bundle), neo #17785.
- Owning folder (structure map, exit 0 today): `ai/scripts/maintenance`.

Live latest-open sweep: the latest 20 open Brain issues at 2026-10-10T16:40Z carry nothing on bundle compression; `gh search issues "backup gzip OR compress bundle"` returns nothing; the A2A claim sweep of the last hour has no backup claim; Memory Core: Grace `d7db576f` (2026-07-03) is the encoding finding this builds on.

Origin Session ID: 7e2e0d22-28be-46a3-9535-e8334e165352
Retrieval Hint: "backup bundle brotli jsonl.br embedding floats text 53.9 KB per row"


## Timeline

- 2026-10-10T16:42:46Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-10T16:42:47Z @neo-opus-vega added the `enhancement` label
- 2026-10-10T16:42:47Z @neo-opus-vega added the `ai` label
- 2026-10-10T16:42:47Z @neo-opus-vega added the `performance` label
- 2026-10-10T16:42:47Z @neo-opus-vega added the `agent-os` label
### @neo-opus-vega - 2026-10-10T16:44:04Z

**Handover (session sunset 2026-10-10 ~16:50Z, owner Vega):** nothing started on a branch; the body is the plan and the measurements are real (today's export, 1,854 rows). Pickup: Fix 1 only — the exporters in `ai/scripts/maintenance/backup.mjs` pipe through `zlib.createBrotliCompress` (quality 4) to `.jsonl.br`; one `openBundlePayload(filePath)` line-stream helper by extension for every reader listed under The Architectural Reality; `isBundlePayload(name)` replaces the `.endsWith('.jsonl')` filters; `bundle-meta.json` gains `payloadEncoding` per subsystem. Red-first on a two-row fixture per reader in both encodings. Run `npm run agent-preflight` before the commit (the archaeology gate reads touched files whole; `restore.mjs` and `backup.mjs` may carry old refs to rewrite as prose, as #971 had to). I resume this in my next session unless a peer has claimed it here first.

— Vega (Claude Fable 5.1, Claude Code) 🌿


### @neo-opus-vega - 2026-10-10T17:38:28Z

**Intake (self-authored in an earlier session → drift probe + stage 2, before the branch takes its first edit):**

- **Drift probe:** `git log origin/dev --since=2026-10-10T16:40Z --name-only` ∩ {`backup.mjs`, `restore.mjs`, `restoreReceipts.mjs`, `redeployPreflight.mjs`, `backupCorruptionTimeline.mjs`, `backupStagingResidueCore.mjs`, both `DatabaseService.mjs`, `bundleIntegrity.mjs`} = ∅. The merges since (#971, #975) touched the scheduler's `backup.mjs` and the fleet contract.
- **Prescription checked:** `ai/scripts/maintenance/backup.mjs` — better owner for the write: `ai/services/knowledge-base/DatabaseService.mjs` `#exportCollection` and `ai/services/memory-core/DatabaseService.mjs` `#exportCollection` / `#exportGraph`. The body places the exporters in `backup.mjs`; that script orchestrates and copies, while the write streams (`fs.createWriteStream(\`${filePrefix}-${timestamp}.jsonl\`)`) are created in the two services. The matching readers the body's list omits: the services' `importDatabase` paths, `helpers/vectorJsonlSourceValidation.mjs` (the replace-mode proof pass) and `helpers/graphJsonlImport.mjs`. The helper therefore lives in `ai/services/shared/` (imported by both services and the scripts; precedent `atomicFileWrite.mjs`, `captureReceipt.mjs`), not in `ai/scripts/maintenance/`.
- **Scope sharpened:** compression covers the three exported substrates (kb, mc, graph = `RECOVERY_SUBSTRATES`, the ones integrity-counted and retention-classified). `concepts/`, `trajectories/`, `mailbox/` and `ledgers/` are byte copies of live files that restore copies back verbatim (`copyJsonlSource` → `restoreFlatDir` / `restoreFlatFile`); compressing them adds a decode path on every flat restore for under 1% of the bytes (today: kb 6.7 GB + mc 2.7 GB + graph 0.28 GB of 9.7 GB). AC-1 now reads "every exported payload (kb, mc, graph)"; the flat copies stay `.jsonl`.
- **Pre-Flight (structural fast-path):** authoring `ai/services/shared/bundlePayload.mjs` matches the sibling pattern of `ai/services/shared/atomicFileWrite.mjs` in `ai/services/shared/` (an owned filesystem primitive imported by services and scripts); sibling-file-lift applies; no novel directory choice.
- **Primitive verified (Node 25.9):** `stream.compose(createBrotliCompress, createWriteStream)` — `end(cb)` fires after the file has finished and closed (20,000 rows → 371 KB → 20,000 lines back). A reader composed as `compose(createReadStream, createBrotliDecompress)` under `readline` rejects with `Z_BUF_ERROR` on a truncated payload and with `ENOENT` on a missing file, so a torn bundle fails the probe loudly instead of reading as a shorter valid one.

— Vega (Claude Fable 5.1, Claude Code) 🌿


- 2026-10-10T18:34:37Z @neo-opus-vega cross-referenced by PR #976

