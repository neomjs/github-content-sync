---
id: 1700
title: Run the turn-document migration on the plane and record the reclaim
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-14T08:33:53Z'
updatedAt: '2026-11-14T09:14:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1700'
author: neo-opus-ada
commentsCount: 2
parentIssue: 133
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
# Run the turn-document migration on the plane and record the reclaim

## Context

#1441 (PR #1457, merged 2026-11-03) shipped `ai/scripts/migrations/dropTurnDocuments.mjs`. It clears the stored document of existing turn rows in Memory Core's memory collection. Since #128 (PR #1447), the write side no longer stores that document.

So far the migration has run only against the run-scoped test server and a scratch server: L3. Its plane run and its byte measurement were deferred as `[L4-deferred — operator handoff needed]` on #1441, and #1457 named #133 as their Residual-Owner. Nothing tracked them after that. #133's epic-resolution review ([comment 6874557400](https://github.com/neomjs/neo-agent-brain/issues/133#issuecomment-6874557400)) found this the one untracked residual, and it is the reason the epic cannot close.

**Observed (2026-11-14, ~08:30Z):** read-only `GET`s against the plane's Chroma (`127.0.0.1:8000`) show `neo-agent-memory` holding 59,491 rows. **Unknown:** how many qualify and how many document bytes they hold. Step 1's dry-run reports both.

## The Problem

Until the migration runs on the plane, every qualifying turn row stores its text twice: once as the document, once as the split `prompt` / `thought` / `response` fields. #128 exists to remove that redundancy. The epic's reclaim stays unmeasured, and #133 stays open.

The dry-run only reads. `--apply` mutates the live store, so it waits for the operator's explicit yes in chat. Every batch is snapshotted first.

## The Architectural Reality

- **The script's modes** (`ai/scripts/migrations/dropTurnDocuments.mjs`):
  - default dry-run: counts, writes nothing;
  - `--apply --snapshot <file>`: appends each batch's prior documents to the snapshot and flushes it to disk before the update;
  - `--restore <file>`: writes a snapshot back.
  - `main()` opens the collection through `ChromaManager.ready()` → `getMemoryCollection()`. It prints `{rows, drop, mismatch, empty, other, cleared, documentBytes}`.
- **Which rows qualify:** `isReconstructableTurn` in `ai/services/memory-core/helpers/turnDocumentText.mjs`. A cleared row keeps its stored vector, because the reconstruction is byte-identical. #133's Avoided Trap: never re-embed during the backfill.
- **Disk:** SQLite does not shrink without a VACUUM. #1432 owns that step. It needs temporary space up to the post-migration file size and runs on a copy.
- **#133's Avoided Traps that bind this run:** never measure or migrate on the live `chroma.sqlite3` file (it uses a rollback journal); copy paths (defrag, restore, import, heal) must not reintroduce documents.
- **Structure map:** N/A. The run touches no file placement.

## The Fix

The run, in order:
1. **Dry-run** against the plane. Post the printed JSON here, with the Brain `dev` SHA it ran from.
2. **Snapshot target:** a disk with room for about `documentBytes`. Name the path and the free space.
3. **Apply**, after the operator's explicit yes: `--apply --snapshot <file>`. Keep the snapshot until AC-3 passes.
4. **No-regression read** over migrated rows. Memory search, session summarization and turn display should each still show the turn's prompt, thought and response.
5. **Record the logical reclaim** (the cleared rows and their `documentBytes`) on #133. The file-size reclaim follows #1432's VACUUM.

## Acceptance Criteria

- [ ] AC-1: The dry-run's JSON is posted here with its Brain SHA and the snapshot disk's free space.
- [ ] AC-2: `--apply --snapshot` ran after the operator's explicit yes. `cleared` equals the dry-run's `drop`, or the difference is explained. The snapshot's path and size are recorded.
- [ ] AC-3: Each read renders the migrated turns' prompt, thought and response, recorded with the ids it read:
  - a `query_raw_memories` hit;
  - a `query_recent_turns` read with `detail: full`;
  - a session summary over migrated rows.
- [ ] AC-4: The cleared rows and `documentBytes` are recorded on #133. Its matrix row names #1432 as the owner of the file-size reclaim.

## Out of Scope

- **The VACUUM and the file-size measurement:** #1432.
- **`kb-restore-20260806`** (59,754 rows), the August restore copy: #1416 counts it, and the decision to drop it is #66's.
- **Code changes.** If the dry-run finds a defect, it gets its own ticket.

## Avoided Traps

- **Applying without a snapshot.** The script refuses `--apply` without `--snapshot`, and the snapshot is the only undo.
- **Reading file size off the live store.** Measure on a copy (#1432).

## Related

Parent: #133 · Code: #1441 / PR #1457, #128 / PR #1447 · Disk: #1432 · Adjacent: #1416, #66

Live latest-open + closed sweep:
- Checked the latest 20 open Brain issues (#1512–#1695) and a closed search (`dropTurnDocuments`, "turn documents", "stored document"). The closed hits were #1441, #128 and #1496 (not planned, a different surface). Done at 2026-11-14T08:35Z; no equivalent or prior exclusion found.
- Backlog: 152 open, red band. This folds in as #133's own sub.

A2A in-flight claim sweep: the 30 newest lane-claims through 08:28Z; none on this scope.

MC sweep: `query_raw_memories` on the symptom; the only hit is the 11-03 handoff of #1457's post-merge steps.

Own-assignment sweep: my four open Brain epics. #133 is the parent.

Ownership: @neo-opus-ada. The `--apply` step waits for the operator's yes. No `good first issue` / `help wanted`: the run needs plane access and that yes.

Origin Session ID: 0f78c55e-2eb9-4b92-a44a-39ccc1af5a42
Retrieval Hint: "dropTurnDocuments plane run reclaim #133 residual #1457"

## Timeline

- 2026-11-14T08:33:54Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-14T08:33:55Z @neo-opus-ada added the `enhancement` label
- 2026-11-14T08:33:55Z @neo-opus-ada added the `ai` label
- 2026-11-14T08:33:55Z @neo-opus-ada added the `agent-os` label
- 2026-11-14T08:34:04Z @neo-opus-ada added parent issue #133
- 2026-11-14T08:34:44Z @neo-opus-ada cross-referenced by #133
### @neo-opus-ada - 2026-11-14T09:12:31Z

**AC-1: the dry-run, 2026-11-14 ~09:12Z.** Run inside the plane's Memory Core container, the documented in-container path: `docker exec neo-local-agent-os-mc-server-1 node ai/scripts/migrations/dropTurnDocuments.mjs`. That is Brain `0a1f4cf` at `/app`, using the plane's own config.

```json
{
  "collection": "neo-agent-memory",
  "mode": "dry-run",
  "rows": 59504,
  "drop": 54836,
  "mismatch": 0,
  "empty": 4620,
  "other": 48,
  "cleared": 0,
  "documentBytes": 143526029
}
```

**What it says:**
- 54,836 of 59,504 turn rows qualify, holding 143.5 MB of document text.
- 0 mismatches, so every qualifying row's stored document equals its reconstruction.
- 4,620 rows are already empty, which matches the write side #128 shipped.
- Nothing was written.

**The snapshot disk, and a blocker beside it:**
- The container's persistent volume (`/app/.neo-ai-data`, the Docker VM disk) has **1.7 GB free of 147.1 GB (99% used)**. The host disk has 95 GiB free.
- The `--apply` snapshot (about 144 MB) fits in what is left.
- #1432's VACUUM does not fit: it needs temporary space up to the post-migration size of `chroma.sqlite3`, which was 4.47 GB on 11-02.
- The full disk is a risk for the whole plane, not only this run.
- `docker system df` reports about 64 GB reclaimable: 31.0 GB of images, 31.9 GB of unused local volumes and 1.6 GB of build cache.
- Pruning, or growing the VM disk, is the operator's call. I am raising it with him.

**Next:** AC-2 (`--apply --snapshot`) waits on the operator's yes, and the VACUUM on disk headroom.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

### @neo-opus-ada - 2026-11-14T09:14:28Z

**The disk blocker, with the prune decision recorded here.** I took the inventory read-only and deleted nothing.

**Old plane-cut images:**
- 11 older tags per service, across 4 services: 44 images, 13.4 GB unique.
- Kept: `latest`, the current cut, and `rollback-20261027-0fa6deb8`, the newest rollback.

**Other unused images:** test, proof, fixture and base images, about 17 GB (`docker system df`: 31 GB reclaimable for images in all).

**Unused volumes:** 16, about 32 GB.
- The largest, 24.5 GB, belongs to a separate environment and stays unless the operator says otherwise.
- So does a 3.3 GB copy.
- The rest are 4 anonymous volumes (3.2 GB) and test data.

**A safe set:** the 44 old plane images, the unused test and base images, the anonymous and test volumes, and the build cache. That frees about 34 GB, taking `/app/.neo-ai-data` from 1.7 GB free to about 35 GB, which leaves room for #1432's VACUUM.

**Decision:** prune that set, or grow the Docker VM disk. The operator has the exact list. #1700's `--apply` waits on his yes either way.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


