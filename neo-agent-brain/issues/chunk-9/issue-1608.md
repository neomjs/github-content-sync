---
id: 1608
title: '--clear-backoff looks the streak up under a key the sweep never writes'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-11T18:57:05Z'
updatedAt: '2026-11-12T16:37:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1608'
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
closedAt: '2026-11-11T19:53:19Z'
---
# --clear-backoff looks the streak up under a key the sweep never writes

## Context

Found on the local plane on 2026-11-11 while clearing a tenant's backoff after a cut (the post-merge step of #1597). The clear reported success and cleared nothing; the tenant stayed suppressed until its own retry.

## The Problem

`TenantRepoSyncService#clearTenantRepoBackoff` reads the persisted state as `revisions[slug]`, where `slug` is the configured `repoSlug`. The sweep writes that state under `${repo.tenantId}/${repo.repoSlug}`. On a real manifest the clear therefore finds no entry for any repo, reports `no-persisted-state` under "already clear", exits 0, and leaves the streak in place.

Reproduced on the local plane at Brain `7219f4e1`, 2026-11-11 17:45Z, inside the orchestrator container:

- The daemon, every cycle: `neo-shared/github-content-sync suppressed by backoff (next ~2026-11-11T19:03:49.221Z, consecutiveFailures=35, …)`.
- `syncTenantRepos.mjs --repo-slug github-content-sync --clear-backoff`: exit 0, `cleared 0 of 1 selected repo(s); 1 already clear`, with `{"repoSlug":"github-content-sync","consecutiveFailures":0,"reason":"no-persisted-state"}`.
- `--repo-slug neo-shared/github-content-sync --clear-backoff`: exit 3, not configured; the configured slugs are the bare ones.
- The manifest (`tenant-repo-sync-revisions.json`, the `revisions` map read inside the same container at 18:48Z): six entries, every key tenant-qualified (`neo-shared/<slug>`). `neo-shared/github-content-sync` still held `consecutiveFailures: 35` and no `backoffClearedAt`.

Not tested: the form without a selector. It walks the same loop, so I expect the same result.

The unit arm is green because its fixture is one the sweep never writes. `TenantRepoSyncService.spec.mjs` ~:296-304 keys the manifest by the bare slug (`'acme/throttled'`) while it configures `tenantId: 't1'`; the sweep's own arms key theirs `t1/<slug>` (~:537, ~:602, ~:709).

History: the clear path came with `neomjs/neo#17067` (PR `neomjs/neo#17352`). #1411 (PR #1427) made it resolve the configured repositories. Its post-merge criterion asked for `completed` or `already-clear`, never `repo-not-configured`; the run above meets that wording while clearing nothing.

## The Architectural Reality

- `ai/daemons/orchestrator/services/TenantRepoSyncService.mjs` owns both sides. The sweep's key is `${repo.tenantId}/${repo.repoSlug}` at ~:2440, ~:2475 and ~:2512, and the module already has `createTenantRepoAccessKey(repo)` (~:310), which returns the same string: "the stable internal key for one effective tenant repository".
- The clear, `clearTenantRepoBackoff` (~:4078), builds `targetSlugs` from `repo.repoSlug` and indexes `revisions[slug]` (~:4113, ~:4126).
- `ai/scripts/maintenance/syncTenantRepos.mjs` passes `--repo-slug` through as `onlyRepoSlugs`. It needs no change.
- Design authority: the method's own JSDoc. "Repos with no persisted entry, or already at zero, are reported as `unchanged` rather than silently succeeding: an operator who clears a backoff and is told nothing has no way to tell a completed no-op from a misspelled selector, which is the ambiguity this whole path exists to remove."
- Structure map (`npm run ai:structure-map -- --files --loc`): both files are listed where they are. No file is added or moved.

## The Fix

In `clearTenantRepoBackoff`, select the configured repo entries, not their slugs, and read and write the manifest under `createTenantRepoAccessKey(repo)`. The outcome rows keep `repoSlug` and add `tenantId`, so two tenants with one slug stay distinguishable. The sweep's three inline copies of the key call the same function, so the two sides cannot drift again.

Rewrite the clear arm's fixture to the keys the sweep writes.

## Contract Ledger

| Target surface | Source of authority | Proposed behaviour | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `TenantRepoSyncService#clearTenantRepoBackoff` result (`details.cleared[]`, `details.unchanged[]`) and the lines `syncTenantRepos.mjs --clear-backoff` prints from it | this ticket | a selected repo's streak is found under its tenant-qualified key and reset; each outcome row carries `tenantId` beside `repoSlug` | a slug configured for two tenants clears both; a repo the sweep never persisted still reads `no-persisted-state` | the method's JSDoc | unit arms; the post-merge receipt |

Surface anchor: `clearTenantRepoBackoff` at ~:4078 and `createTenantRepoAccessKey` at ~:310, read at `7219f4e1`.

Decision Record impact: none.

## Acceptance Criteria

- [ ] A unit arm writes the manifest with the keys the sweep writes (`<tenantId>/<repoSlug>`), clears by `repoSlug`, and the streak is zero with `backoffClearedAt` and `backoffClearedFromFailures` set. Red on `dev` first.
- [ ] `isRepoDue` flips from suppressed to due for that entry, as the existing arm asserts.
- [ ] A slug configured for two tenants clears both entries, and each outcome row names its tenant.
- [ ] The sweep and the clear take the manifest key from one function.
- [ ] (post-merge, after a plane cut that carries it) On the local plane, `--clear-backoff --repo-slug <slug>` on a suppressed repo reports it under `cleared` with its previous streak, and the next sweep attempts it. A maintainer posts the receipt here.

## Out of Scope

- Whether a deploy should clear streaks by itself (defect-note of 2026-11-11: a cut that fixes a tenant's failure cause leaves its streak in place).
- The stale `neo-shared/neo` entry in the local manifest (last attempt 2026-08-06, 5 failures).
- Backoff and recovery semantics (#1382).

## Avoided Traps

- **Accepting `<tenantId>/<slug>` as the selector.** The documented bare-slug form would keep reporting green while clearing nothing.
- **Editing the manifest by hand.** A running daemon owns that file.

## Related

#1411 · #1597 · `neomjs/neo#17067`

Live latest-open + closed sweep: latest 20 open of 149 (red band; no open ticket owns the clear path, and #64 was read: it covers scheduling) and closed search (`clear-backoff backoff streak`; #1411 read with its Out of Scope) at 2026-11-11T18:55Z; no equivalent found. A2A in-flight claim sweep: latest 30 messages, all read states, at 2026-11-11T18:55Z; no claim on this surface. MC sweep: "clear-backoff reports already clear while the tenant repo stays suppressed by backoff", 6 results, no prior decision. Own-assignment sweep: 15 open, #64 read, none overlapping.

Origin Session ID: 20e7f7fd-c32d-4575-92d1-fd9020bf5c31

Retrieval Hint: "clear-backoff no-persisted-state tenant-qualified manifest key createTenantRepoAccessKey"

— Vega (Claude Fable 5.2, Claude Code) 🌿

## Timeline

- 2026-11-11T18:57:05Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-11T18:57:07Z @neo-opus-vega added the `bug` label
- 2026-11-11T18:57:07Z @neo-opus-vega added the `ai` label
- 2026-11-11T18:57:07Z @neo-opus-vega added the `agent-os` label
- 2026-11-11T18:57:44Z @neo-opus-vega cross-referenced by #1411
- 2026-11-11T19:21:10Z @neo-opus-vega cross-referenced by PR #1611
- 2026-11-11T19:21:39Z @neo-opus-vega cross-referenced by #64
- 2026-11-11T19:48:04Z @neo-opus-vega cross-referenced by #1612
- 2026-11-11T19:53:19Z @tobiu referenced in commit `c27e8b9` - "fix(orchestrator): --clear-backoff finds the streak under the sweep's key (#1608) (#1611)

clearTenantRepoBackoff read revisions[repoSlug], while the sweep writes
revisions[tenantId/repoSlug]. On a real manifest it found no entry,
reported no-persisted-state and exited 0 with the streak in place. It
now keys by createTenantRepoAccessKey(repo), the function the sweep's
six key sites call too, and each outcome row names its tenant.

The clear arms' fixtures were keyed by the bare slug, a shape no sweep
writes. They use the sweep's keys now, and one arm covers a slug that
two tenants configure."
- 2026-11-11T19:53:20Z @tobiu closed this issue
### @neo-opus-vega - 2026-11-11T20:49:35Z

**Handover for the post-merge criterion (AC-5), 2026-11-11 (owner Vega, session `20e7f7fd-c32d-4575-92d1-fd9020bf5c31`).**

- **State:** PR #1611 merged as `c27e8b9b`. The local plane still runs `7219f4e1`, so the fix is not deployed there.
- **Why no cut today:** the installed Fleet Manager is paired with that plane revision, and FM row 3 walks the pairing on 2026-11-12 at 14:00Z. I propose the cut to Ada after that sitting.
- **Pickup:** after a cut that carries `c27e8b9b`, find a suppressed repo (`suppressed by backoff … consecutiveFailures=N` in the orchestrator log; a cut usually kills one running attempt and leaves one). Run `syncTenantRepos.mjs --clear-backoff --repo-slug <slug>` inside the orchestrator container and post here: the command's output, with the repo under `cleared`, its tenant and its previous streak, and the next sweep's `Refreshing <tenant>/<slug>` line.
- **Related:** #1612 covers the two readers that still build the key by hand. The residual is registered on #64 (comment 6533628165).

— Vega (Claude Fable 5.2, Claude Code) 🌿

### @neo-opus-vega - 2026-11-12T14:29:55Z

**AC-5 receipt status after the 2026-11-12 cut (plane on `cfb998af`): not taken yet.** The cut left no fresh streak: `tenant-repo-sync-revisions.json` shows 0 failures for create-app, all three devindex tenants and github-content-sync, each attempted today. The only suppressed repo is `neo-shared/neo` (5 failures, last attempt 2026-08-06, `KB_VECTOR_EMBED_FAILED`, never ingested). Clearing it would start that repo's first full ingest rather than release a lane the cut interrupted, so I left it and logged a defect-note. The receipt waits for the next cut that leaves a working tenant with a streak; Residual-Owner #64 stays.

**Update 16:37Z:** the 15:59Z cut to `528105e7` also left none: all five working tenants read `consecutiveFailures=0`, and `neo-shared/neo` is unchanged at 5. `syncTenantRepos.mjs` has no dry-run, so the lookup cannot be shown without clearing.

— Vega (Opus 5.5, Claude Code) 🌿


