---
id: 29
title: The data-sync pipeline has published nothing since 2026-08-30
state: CLOSED
labels:
  - bug
  - ai
  - architecture
  - github_actions
assignees:
  - neo-opus-ada
createdAt: '2026-09-24T11:08:31Z'
updatedAt: '2026-09-24T13:41:54Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/29'
author: neo-opus-ada
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
closedAt: '2026-09-24T13:41:54Z'
---
# The data-sync pipeline has published nothing since 2026-08-30

## Context

The data-sync pipeline (`.github/workflows/data-sync-pipeline.yml`) has failed on every run since 2026-08-30 20:42Z: 100 of the last 100, on an unchanged commit. The last success was 16:14Z that day. Each run collects (opt-in, opt-out, spider, updater), then stops at **Probe publish access**, before **Publish the working set**. So every two hours, a run's work is discarded. Nothing has flagged it for 25 days.

I found this while starting #27. That deploy needs a non-empty contributor stream from a source that survives the `pages` rebuild (neomjs/neo#19047).

## The Problem

Measured 2026-09-24:

1. **The middleware's rsync deleted the published set.** The scheduled middleware rebuild runs `gcloud storage rsync --recursive --checksums-only --delete-unmatched-destination-objects dist gs://neomjs-middleware-dist/dist`. Its local `dist/` has no `devindex/`. At 2026-08-30 20:03Z it removed all 22 objects under `dist/devindex/`, the 12 dated `archive/users-*.jsonl` restore copies among them. From its run log, `logs/scheduled/run-2026-08-30T19-45-49-407Z.log`, which is untracked on the host that runs it:
   ```
   STEP: rsync dist -> GCS — gcloud storage rsync --recursive --checksums-only --delete-unmatched-destination-objects dist gs://neomjs-middleware-dist/dist
   Removing gs://neomjs-middleware-dist/dist/devindex/users.jsonl#…
   Removing gs://neomjs-middleware-dist/dist/devindex/working-set-manifest.json#…
   ```
   Two writers share one prefix, and one of them deletes whatever it did not write.
2. **The probe cannot pass on an empty prefix.** `gcloud storage ls "${DEVINDEX_PUBLISH_BUCKET}/"` answers `One or more URLs matched no objects.`: the listing succeeded and found nothing. The step reports every non-impersonation failure as *"Authenticated, but cannot list the destination — the bucket grant, not the identity"* and exits 1. The publish that would refill the prefix runs after the probe, so the pipeline cannot recover by itself. The first failing run (33334388654) and the latest (35978993043) end on the same two lines.
3. **Nothing reads the bucket anyway.** `config.publishedWorkingSet.baseUrl` is still `https://neomjs.com/node_modules/neo.mjs/apps/devindex/resources/data/`. That URL serves the copy committed in `pages`, not the bucket. The copy is dated 2026-08-31: `users.jsonl` is 24,084,949 bytes and 49,999 records, byte-identical to `pages` `main@1847ca65`. The bucket is private (403) and no middleware route maps it. `Storage#fetchManifest`'s docblock already says verification "is still not live". So even before 08-30, runs hydrated from the frozen copy and published to a store no reader used, and the sync cursors reset on every run.
4. **Every stage re-adopts the published set.** *(Found 2026-09-24 while implementing.)* Hydration is memoized per process, and the workflow runs each stage as its own `npm run`. Run 35978993043 adopts the set three times (09:04:47Z, 09:04:51Z, 09:09:47Z). Each adoption overwrites what the stages before it wrote, so only the last stage's writes can reach the publish: an opt-out recorded by OptOut is gone before Spider starts. No opt-out has arrived yet (the intake repository has 0 stars and no issues since 2026-08-18), so the defect is latent.

**Consequences:**
- The live DevIndex shows 08-31 data.
- An opt-out processed now is confirmed to the requester and then discarded. None has arrived since the last successful publish: `devindex-opt-out` has no stars and no closed issues since then.
- When `pages` redeploys against 13.2 without its devindex copy (neomjs/pages#7), `baseUrl` stops answering, and `pullDevIndexData` leaves every checkout with an empty grid.
- `pages` also serves the pipeline state publicly today, `blocklist.json` included.
- The seed survives in `pages`' git history at `1847ca65`, all nine files.

## The Architectural Reality

- `buildScripts/publishWorkingSet.mjs` writes the members, then the manifest, to `DEVINDEX_PUBLISH_BUCKET` = `gs://neomjs-middleware-dist/dist/devindex` (a repository variable).
- The Cloud Run middleware serves `gs://neomjs-middleware-dist/dist` as its `/app/dist`. Its unattended 8-hour rebuild, `buildScripts/scheduledBuild.mjs` on neomjs/middleware-v2 branch `clio/rsync-maxbuffer`, mirrors its own `dist/` over that prefix with delete.
- `Storage#hydrateWorkingSet` fetches the nine working-set members (`Storage#workingSetMembers`) from `baseUrl`. When no manifest answers, it adopts them unverified.
- neomjs/neo D#19050 (C′, graduated 2026-09-23) gives devindex its own Pages site (#27). That is the "when the destination is chosen" moment that `baseUrl`'s docblock waits for.

## The Fix

The pipeline's system of record moves out of the middleware's delete scope, and the pipeline reads back what it publishes.

*Updated 2026-09-24 12:10Z: option A agreed by @neo-opus-grace; the public read is split out (see the plan comment).*

1. **Destination:** `gs://neomjs-middleware-dist/devindex`, outside `dist/`. The publish identity's grant is bucket-level (`setup-gcp-publish.sh`: `roles/storage.objectAdmin` on the bucket, no condition), so the one operator step is setting `DEVINDEX_PUBLISH_BUCKET`.
2. **Probe:** a destination the identity can list but that is empty counts as reachable. Only a refused listing fails.
3. **Hydration** reads the store it publishes to, authenticated and with the manifest digest check, instead of the public `baseUrl`. That makes verification live.
4. **Seed:** while the store has no manifest (a definite 404), hydration falls back once to the public 2026-08-31 copy, pinned at `pages@1847ca65` so the `pages` rebuild cannot remove it. The first green publish ends the fallback, and the log names the source used. Any other store failure adopts nothing.
5. **Once per run:** the first process of a workflow run hydrates and marks the run; the later stages keep the local set.

The public read, where the Pages site (#27) and `devindex:pull-data` take `users.jsonl` from the published set and a green run redeploys the site, follows as its own ticket once #27's workflow is on `dev`.

Fork, for the pipeline's author (@neo-opus-grace, neomjs/neo#17375), decided: **A**.

| option | falsifier |
|---|---|
| **A (recommended):** a private store as the system of record, with the Pages site as the public read | a run's published manifest digests do not match what the next run hydrates |
| B: the Pages site as the only store (a publish is a deploy), with GCS retired | the opt-out state becomes public; rejected unless it can stay out of the site |
| C: keep `dist/devindex` and add `--exclude` to the middleware rsync | every future middleware sync path must remember the exclusion, and forgetting it fails as silent deletion again |

## Acceptance Criteria

- [ ] A scheduled run is green end to end, and its publish lands in the new destination.
- [ ] The next run hydrates that set with the digests verified. Its log has no `adopting the fetched set UNVERIFIED`.
- [ ] One scheduled middleware rebuild leaves the set intact. Receipt: the destination's object listing before and after.
- [ ] The probe passes on an empty destination and fails on a refused one. Both arms are exercised.
- [ ] The first run against the empty store seeds from the public copy, and says so in its log.
- [ ] A run adopts the set once: the later stages keep the earlier stages' writes, and the run log shows one adoption.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| Store source: `Storage#workingSetSource()`, fed by the job's `DEVINDEX_PUBLISH_BUCKET` (repository variable) and each stage's `DEVINDEX_STORE_TOKEN` (the `gcp-auth` step's `access_token`) | Fix 1, 3 | With both set, the working set is read from `https://storage.googleapis.com/<bucket path>/` with `Authorization: Bearer <token>` | Either unset: the public copy, `config.publishedWorkingSet.baseUrl` (developers, tests) | JSDoc | `StorageWorkingSetHydration.spec.mjs`: "a run holding the store credentials reads the store…"; `DataSyncPublishProbe.spec.mjs`: "the probe and the store token come before every stage that hydrates" |
| The working set: `Storage#workingSetMembers()`, nine members (`users`, `tracker`, `visited`, `blocklist`, `allowlist`, `threshold`, `failed`, `optinSync`, `optoutSync`) | Fix 3 | Fetched together and written only after every member arrived, all-or-nothing | Any member non-2xx or throwing: nothing is adopted, and the local set is unchanged | JSDoc (`hydrateWorkingSet`, `config.publishedWorkingSet`) | spec: "a non-2xx on any member adopts NOTHING…", "a member that throws adopts NOTHING…" |
| Manifest `working-set-manifest.json` `{digests: {<key>: sha256}}`: written by `Storage#recordWorkingSetManifest()`, published last by `publishWorkingSet.mjs` | Fix 3, 4 | A store read requires HTTP 200 with `digests`, and every member must match its digest | Store 404: seeded once from the public copy, unverified, and the log says so. Any other status, or a 200 without `digests`: nothing is adopted | JSDoc | spec: matching-digest control, digest mismatch, "a store that never published is seeded once…", "a store that fails to answer adopts nothing…", "a store that answers without digests adopts nothing…" |
| Run mark `hydrated-run.txt` (`config.paths.hydratedRun`) `{run: "<GITHUB_RUN_ID>-<GITHUB_RUN_ATTEMPT>", users: <index size>}`, written by `Storage#hydrateOncePerRun()`; `Manager` hydrates before any command runs | Fix 5 | The first process of a workflow run adopts and marks the run. Its later processes keep the local set, so the earlier stages' writes survive | Outside a workflow run (no `GITHUB_RUN_ID`), every process hydrates. The mark is local and never published | JSDoc | spec: "the first process of a workflow run marks the run…"; `WorkingSetRunBoundary.spec.mjs`: the opt-out and opt-in-removal arms through the real CLI |
| Collapse guard: `publishWorkingSet.mjs#assertNotCollapsed()` | RA-3 on #32 | Refuses to publish when `Storage#countIndex()` < 80% of `Storage#runStartCount()` | No mark for this run: refused, since there is no baseline. `DEVINDEX_ALLOW_INDEX_COLLAPSE` skips the check | JSDoc | `WorkingSetRunBoundary.spec.mjs`: the run-start, drop and no-mark arms |
| Probe step "Probe publish access" (`data-sync-pipeline.yml`) | Fix 2 | A listing that succeeds, including one that "matched no objects", counts as reachable | A refused listing fails as the bucket grant; a refused impersonation fails as the identity; an unset destination fails first | workflow comment | `DataSyncPublishProbe.spec.mjs` (runs the workflow's own script) |

Rows anchored at `db8be16`: `workingSetMembers()` at `Storage.mjs:588`, the store and 404 branches in `fetchAndAdoptWorkingSet()`, `Manager.mjs:139`, and the probe and token lines in the workflow.

## Post-Merge Validation

Operator step at merge: set `DEVINDEX_PUBLISH_BUCKET` to `gs://neomjs-middleware-dist/devindex`. The pre-merge witnesses are unit arms: the probe classification runs the workflow's own script, and hydration covers the store read, the seed, the refusal and once-per-run. The first two scheduled runs after merge then witness the rest, recorded on #33, which owns these receipts once this ticket closes: run 1 seeds, publishes and logs one adoption; run 2 hydrates the store verified; and a middleware rebuild between them leaves the destination's listing unchanged.

## Out of Scope

- The Pages workflow itself (#27), and the public read that wires it and `devindex:pull-data` to the store: its own ticket once #27's workflow is on `dev`.
- Exclusion hygiene in the middleware's rsync (option C).
- The cursor defect in #9.

## Related

#27 · neomjs/neo#19047 · neomjs/neo#19050 · neomjs/neo#17375 · neomjs/pages#7 · neomjs/middleware-v2#19

Sweeps:
- Live latest-open sweep at 2026-09-24T11:05:53Z: this repo (4 open) and neomjs/middleware-v2 (8 open), no equivalent.
- A2A sweep: the last 30 messages, no claim.
- MC sweep: `query_raw_memories("devindex contributor data stale not updating; users.jsonl published working set bucket; hydrateWorkingSet adopting the fetched set UNVERIFIED")`, 6 results, no prior decision.
- Own-assignment sweep: #27 only, which consumes this rather than overlapping it.

Owner: @neo-opus-ada (assigned 2026-09-24; @neo-opus-grace passed the build after agreeing option A).

Origin Session ID: 101d2ce9-9f43-4f5a-9ae2-3c75bf8f6fcf
Retrieval Hint: "devindex data-sync probe matched no objects rsync delete-unmatched-destination-objects dist/devindex"

Authored by ⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code





## Timeline

- 2026-09-24T11:08:33Z @neo-opus-ada added the `bug` label
- 2026-09-24T11:08:33Z @neo-opus-ada added the `ai` label
- 2026-09-24T11:08:33Z @neo-opus-ada added the `architecture` label
- 2026-09-24T11:08:33Z @neo-opus-ada added the `github_actions` label
- 2026-09-24T11:48:20Z @neo-opus-ada cross-referenced by #19047
- 2026-09-24T12:01:08Z @neo-opus-ada cross-referenced by PR #31
- 2026-09-24T12:04:19Z @neo-opus-ada assigned to @neo-opus-ada
### @neo-opus-ada - 2026-09-24T12:05:45Z

## Implementation plan, option A: one PR for the pipeline plane, the public read after #31

Taken per @neo-opus-grace, who agreed option A and asked for the probe fix first.

**Measured constraint.** `buildScripts/setup-gcp-publish.sh` grants the publish identity `roles/storage.objectAdmin` on the whole bucket, with no condition. So a prefix outside the middleware's rsync scope needs **no IAM change**. The one operator step is the variable's value.

**This PR (resolves #29):**
1. **Destination:** `DEVINDEX_PUBLISH_BUCKET` → `gs://neomjs-middleware-dist/devindex`, outside `dist/`, which the scheduled middleware rsync mirrors with delete. Operator step at merge: set the variable. That's the whole infra change.
2. **Probe:** a destination the identity can list but that is empty passes (`matched no objects` means reachable). Only a refused listing fails, and both arms get exercised.
3. **Auth before the stages.** Hydration then reads the store it publishes to, over the GCS endpoint with the run's short-lived token, with the manifest digests verified. That makes verification live, as `Storage#fetchManifest`'s docblock asks.
4. **Seed without an operator copy.** While the store has no manifest, hydration falls back once to today's public copy (pages' 2026-08-31 set). The first green publish ends the fallback, and the log says which source was used.

**Follow-up, after #31 merges (its own ticket):** the Pages site takes `users.jsonl` from the store, a green collection run triggers the site redeploy, and `devindex:pull-data` reads the public site copy. That carries this ticket's last AC (the site's `users.jsonl` equals the run's published digest), and its ACs move there.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


- 2026-09-24T12:15:57Z @neo-opus-ada cross-referenced by PR #32
- 2026-09-24T12:58:18Z @neo-opus-ada referenced in commit `456f313` - "fix(data-sync): hydrate at the stage boundary, verify every store read, guard against the run's start (#29)

Review R1 (Euclid) found three paths that still acted on the wrong state:

- Hydration was lazy, so a stage's first write could precede it: OptOut
  records an opt-out, then deleteUsers hydrates over it. The Manager now
  hydrates after Storage.ready() and before any command runs.
- A store answering 200 without digests was adopted unverified. Only a
  definite 404 seeds from the public copy; any other digest-less answer
  adopts nothing.
- The collapse guard compared with the pinned public seed. The run marker
  now records the index size the run started from, the publisher compares
  against it, and a run without a mark does not publish.

Source comments describe the nine members and the conditional verification."
- 2026-09-24T13:01:57Z @neo-opus-ada cross-referenced by #33
- 2026-09-24T13:13:23Z @neo-opus-ada referenced in commit `db8be16` - "docs(data-sync): the adoption comments say where verification applies (#29)"
- 2026-09-24T13:40:53Z @neo-opus-ada cross-referenced by #34
- 2026-09-24T13:41:54Z @tobiu referenced in commit `d0bf025` - "Merge pull request #32 from neomjs/ada/29-publish-store

fix(data-sync): the pipeline reads the store it publishes to, once per run (#29)"
- 2026-09-24T13:41:55Z @tobiu closed this issue
- 2026-09-24T14:44:08Z @neo-opus-ada cross-referenced by PR #36
- 2026-09-24T14:46:01Z @neo-opus-ada cross-referenced by #37

