---
id: 29
title: The data-sync pipeline has published nothing since 2026-08-30
state: OPEN
labels:
  - bug
  - ai
  - architecture
  - github_actions
assignees: []
createdAt: '2026-09-24T11:08:31Z'
updatedAt: '2026-09-24T11:08:31Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/29'
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

**Consequences:**
- The live DevIndex shows 08-31 data.
- An opt-out processed now is confirmed to the requester and then discarded. None has arrived since the last successful publish: `devindex-opt-out` has no stars and no closed issues since then.
- When `pages` redeploys against 13.2 without its devindex copy (neomjs/pages#7), `baseUrl` stops answering, and `pullDevIndexData` leaves every checkout with an empty grid.
- `pages` also serves the pipeline state publicly today, `blocklist.json` included.
- The seed survives in `pages`' git history at `1847ca65`, all nine files.

## The Architectural Reality

- `buildScripts/publishWorkingSet.mjs` writes the members, then the manifest, to `DEVINDEX_PUBLISH_BUCKET` = `gs://neomjs-middleware-dist/dist/devindex` (a repository variable).
- The Cloud Run middleware serves `gs://neomjs-middleware-dist/dist` as its `/app/dist`. Its unattended 8-hour rebuild, `buildScripts/scheduledBuild.mjs` on neomjs/middleware-v2 branch `clio/rsync-maxbuffer`, mirrors its own `dist/` over that prefix with delete.
- `Storage#hydrateWorkingSet` fetches `users.jsonl`, `tracker.json` and `visited.json` from `baseUrl`. When no manifest answers, it adopts them unverified.
- neomjs/neo D#19050 (C′, graduated 2026-09-23) gives devindex its own Pages site (#27). That is the "when the destination is chosen" moment that `baseUrl`'s docblock waits for.

## The Fix

The pipeline's system of record moves out of the middleware's delete scope, and the pipeline reads back what it publishes.

1. **Destination:** a prefix the middleware does not mirror, either its own bucket or a prefix outside `dist/`. Operator step: the variable and its IAM binding.
2. **Probe:** a destination the identity can list but that is empty counts as reachable. Only a refused listing fails.
3. **Hydration** reads the store it publishes to, through authenticated `gcloud` and with the manifest digest check, instead of the public `baseUrl`. That makes verification live.
4. **Seed:** copy `pages@1847ca65`'s nine files into the new destination once, before the first run.
5. **Public read:** the Pages site (#27) takes `users.jsonl` from the published set, and a green run triggers its redeploy. `users.jsonl` is the only file that goes public. The rest of the state, `blocklist.json` included, stays private.

Fork, for the pipeline's author (@neo-opus-grace, neomjs/neo#17375):

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
- [ ] After a green run, the Pages site's `users.jsonl` matches that run's published digest.

## Out of Scope

- The Pages workflow itself (#27). This ticket gives it a source.
- Exclusion hygiene in the middleware's rsync (option C).
- The cursor defect in #9.

## Related

#27 · neomjs/neo#19047 · neomjs/neo#19050 · neomjs/neo#17375 · neomjs/pages#7 · neomjs/middleware-v2#19

Sweeps:
- Live latest-open sweep at 2026-09-24T11:05:53Z: this repo (4 open) and neomjs/middleware-v2 (8 open), no equivalent.
- A2A sweep: the last 30 messages, no claim.
- MC sweep: `query_raw_memories("devindex contributor data stale not updating; users.jsonl published working set bucket; hydrateWorkingSet adopting the fetched set UNVERIFIED")`, 6 results, no prior decision.
- Own-assignment sweep: #27 only, which consumes this rather than overlapping it.

unowned-rationale: the pipeline is @neo-opus-grace's design (neomjs/neo#17375), so the fork is hers to take or pass. If nobody claims it, I take it after #27.

Origin Session ID: 101d2ce9-9f43-4f5a-9ae2-3c75bf8f6fcf
Retrieval Hint: "devindex data-sync probe matched no objects rsync delete-unmatched-destination-objects dist/devindex"

Authored by ⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-09-24T11:08:33Z @neo-opus-ada added the `bug` label
- 2026-09-24T11:08:33Z @neo-opus-ada added the `ai` label
- 2026-09-24T11:08:33Z @neo-opus-ada added the `architecture` label
- 2026-09-24T11:08:33Z @neo-opus-ada added the `github_actions` label

