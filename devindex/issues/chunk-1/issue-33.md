---
id: 33
title: 'The Pages site and developers'' pulls read the published store, not the 08-31 copy'
state: CLOSED
labels:
  - enhancement
  - ai
  - github_actions
assignees:
  - neo-opus-ada
createdAt: '2026-09-24T13:01:56Z'
updatedAt: '2026-09-24T15:41:57Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/33'
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
closedAt: '2026-09-24T15:41:57Z'
---
# The Pages site and developers' pulls read the published store, not the 08-31 copy

## Context

#29 makes the data-sync pipeline's private store (`gs://neomjs-middleware-dist/devindex`) the system of record, verified by digests. #27 deploys the app as its own Pages site. Two readers still read the frozen public copy (`config.publishedWorkingSet.baseUrl`, pinned at `neomjs/pages@1847ca65`, 2026-08-31):
- the Pages build, through `devindex:pull-data`;
- every developer's `npm install` postinstall, through the same pull.

So the site shows 08-31 data even after the pipeline publishes again.

This ticket also owns #29's post-merge receipts, which #29 cannot hold once its PR resolves it (review R1 on neomjs/devindex#32, RA-5). They are this ticket's premise: without a live, verified store there is nothing to read.

## The Problem

- The Pages workflow (`.github/workflows/pages.yml`, from #27) pulls the public copy, so a green collection run changes nothing a visitor sees.
- Nothing redeploys the site when the data changes. It deploys only on a push to `dev`.
- `devindex:pull-data` gives developers the 08-31 set indefinitely.

## The Architectural Reality

- The store is private and read over the GCS endpoint with the publish identity's short-lived token (`Storage#workingSetSource`). The Pages workflow can mint the same token: the identity's WIF binding is per repository (`buildScripts/setup-gcp-publish.sh`).
- `users.jsonl` is the only member a browser reads. The other eight, the blocklist and the opt-in/opt-out cursors among them, must never reach the public site.
- `deploy-receipt.json` (#27) already records the data's SHA-256 and record count.

## The Fix

1. The Pages workflow authenticates and takes `users.jsonl` from the store, checked against the store manifest's digest, instead of the public pull. The receipt names the manifest it matched.
2. A green collection run that published triggers the Pages workflow (`workflow_dispatch`, `actions: write`), so the site is at most one run behind the store.
3. `devindex:pull-data` reads the live site's copy (`config.publicSite`), so developers pull current data. *(Delta while implementing: `publishedWorkingSet.baseUrl` itself stays pinned. The store's seed needs all nine members and the site serves only `users.jsonl`, so moving it would break the seed.)*

## Acceptance Criteria

- [ ] #29's receipts, recorded here after it merges and `DEVINDEX_PUBLISH_BUCKET` is set: `[L3-deferred — operator handoff needed]`
  - run 1 seeds (`seeding once from`), publishes under `devindex/`, and logs one adoption;
    **Receipt, run [36014534122](https://github.com/neomjs/devindex/actions/runs/36014534122)** (schedule, `f06cbb4956`, 14:40Z, green). The probe logged `it holds nothing yet` and passed. Opt-In logged `seeding once from …/pages/1847ca65…`, then `Adopted the published working set (9 files)`. Opt-Out, Spider and Updater each logged `hydrated earlier in run 36014534122-1; keeping this run's local writes`, so there was one adoption. The collapse check read `50,000 records (was 50,000)`. The publish copied the nine members and `archive/users-2026-09-24.jsonl`, then the manifest last, to `gs://neomjs-middleware-dist/devindex/`.
  - run 2 hydrates the store verified, with no `UNVERIFIED` line;
  - the destination's listing is unchanged across one scheduled middleware rebuild.
- [ ] After a green collection run, the site's `users.jsonl` digest equals that run's published manifest digest, and the deploy receipt names it. `[L3-deferred — operator handoff needed]`

The two deferred ACs need scheduled runs that outlive this issue's PR. Their receipts go to neomjs/neo#19047, the deploy chain's open parent.
- [ ] Only `users.jsonl` is on the site; none of the other eight members is reachable.
- [ ] `devindex:pull-data` fetches the live site copy.

## Contract Ledger

Anchored at `a583de1`.

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| Pages `data` job (new) and `buildScripts/fetchStoreIndex.mjs` (new) | Fix 1 | The job alone holds `id-token: write` and runs no npm. It fetches `working-set-manifest.json`, then `users.jsonl`, from `gs://…/devindex` with the store token, verifies the SHA-256 against `digests.users`, and uploads the file as the `contributor-index` artifact | No fallback: a manifest 404 (the store has published nothing), a manifest without the digest, a mismatch, or any other non-2xx fails the deploy, and the site keeps its last one | JSDoc and workflow comments | `FetchStoreIndex.spec.mjs` (the verified arm and four refusals); `PagesDeploy.spec.mjs`: "no job that runs npm can mint a store token" |
| Pages `build` job | Fix 1 | Needs `data`, downloads `contributor-index` into `apps/devindex/resources/data`, and runs no `devindex:pull-data` | The assembly still refuses an empty index | workflow comment | `PagesDeploy.spec.mjs`: "the build takes the verified index…" |
| `deploy-receipt.json` fields | Fix 1 | `dataSource` is the store object (`gs://…/users.jsonl`), and `dataPublishedAt` is the manifest's `publishedAt`. `dataDigest` (existing) equals the manifest's `digests.users` | A local assembly records `dataSource: local` and `dataPublishedAt: null` | JSDoc | post-merge: the live receipt after a green publish |
| Data-sync "Redeploy the site" step (new), plus `actions: write` on `collect` | Fix 2 | After "Publish the working set", under the same `if`, `gh workflow run pages.yml --ref dev` with the job's `GITHUB_TOKEN`. `workflow_dispatch` is one of the two events that token may start | A failed publish never reaches the step | workflow comment | `PagesDeploy.spec.mjs`: "every publish dispatches the Pages workflow…" |
| `config.publicSite` (new) and `devindex:pull-data` | Fix 3 | `pull-data` fetches `${publicSite}${SITE_DATA}users.jsonl`, which is the deployed site's copy. `SITE_DATA` is exported by `assemblePagesSite.mjs`, which owns the site layout | `config.publishedWorkingSet.baseUrl` stays pinned, for the store's one-time seed only | JSDoc | a local pull fetched 50,000 records from the live site |

## Out of Scope

- The store and the pipeline themselves (#29).
- The `neomjs.com/devindex/` route (neomjs/middleware-v2#22).

## Related

#29 · #27 · neomjs/devindex#32 · neomjs/neo#19047

Depends on #27's workflow being on `dev` (#31) and #29 being merged (#32).

Sweeps:
- Live latest-open sweep at 2026-09-24T12:58:34Z: this repo has 5 open issues and 3 open PRs, all mine or Grace's. No equivalent.
- A2A: my own last broadcasts name this follow-up as planned; there is no other claim.
- MC: the #29 sweeps (6 results) found no prior decision on the public read.
- Own-assignment sweep: #27 and #29, which this one consumes.

Assignee: @neo-opus-ada.

Origin Session ID: 101d2ce9-9f43-4f5a-9ae2-3c75bf8f6fcf
Retrieval Hint: "devindex public read pages site users.jsonl store digest deploy receipt"

Authored by ⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code





## Timeline

- 2026-09-24T13:01:57Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-24T13:01:57Z @neo-opus-ada added the `enhancement` label
- 2026-09-24T13:01:58Z @neo-opus-ada added the `ai` label
- 2026-09-24T13:01:58Z @neo-opus-ada added the `github_actions` label
- 2026-09-24T13:14:19Z @neo-opus-ada cross-referenced by #29
- 2026-09-24T13:15:06Z @neo-opus-ada cross-referenced by PR #32
- 2026-09-24T13:40:53Z @neo-opus-ada cross-referenced by #34
- 2026-09-24T14:44:08Z @neo-opus-ada cross-referenced by PR #36
- 2026-09-24T14:46:01Z @neo-opus-ada cross-referenced by #37
- 2026-09-24T15:41:57Z @tobiu referenced in commit `8c995bf` - "Merge pull request #36 from neomjs/ada/33-public-read

feat(pages): the site serves the store's latest publish, and every publish redeploys it (#33)"
- 2026-09-24T15:41:58Z @tobiu closed this issue

