---
id: 1959
title: 'The in-memory graph cache never evicts, so mc-server''s heap grows to OOM'
state: CLOSED
labels:
  - bug
  - ai
  - performance
assignees:
  - neo-opus-ada
createdAt: '2026-11-27T17:37:05Z'
updatedAt: '2026-11-27T20:57:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1959'
author: neo-opus-ada
commentsCount: 2
parentIssue: 1843
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-27T19:49:54Z'
---
# The in-memory graph cache never evicts, so mc-server's heap grows to OOM

Owner: Ada (`@neo-opus-ada`), owner of the parent #1843. Operator priority: memory leaks first (2026-11-27).

## Context

mc-server died of JS heap OOM at 11-26 14:33Z, after 18.97 h up, and at 11-27 06:10Z, after 15.61 h. Its next incarnation stopped answering at about 8.2 h, and the cut at 14:33Z replaced it. On the current incarnation (`46228e03`) the heap's old generation grows about 100–150 MiB an hour toward the 2 GB `--max-old-space-size`.

The measurements in [#1843 comment](https://github.com/neomjs/neo-agent-brain/issues/1843#issuecomment-7224583482) were read on the live process through its container-local inspector, and they show the in-memory graph cache is the leak:
- `GraphService.db` held 113,111 nodes and 491,443 edges at 17:15:44Z, and 501,113 edges 11 min later, about +53k edges an hour.
- `lastAccessMap` tracked only 273 entries, and `maxGraphNodes` is `null`.
- A 20-min sampled heap profile puts 60 of the 106 MiB still live at its end in `loadNodeVicinitySync`'s parsing of edge and node rows (`ai/graph/storage/SQLite.mjs:967`, `:983`).

## The Problem

`Database#getAdjacentNodes` (`ai/graph/Database.mjs:340`) loads a node's whole vicinity into the cache: the node, every edge touching it, and every neighbor node, as full JSON. It tracks only the anchor in `lastAccessMap` (`:377`). Three things make that a leak:
- `runGarbageCollector` (`:196`) runs only when `maxGraphNodes` is set, and production leaves it `null` (`:52`).
- Even with a bound, it would evict from the tracked anchors and never from the neighbors a hub's vicinity brought in.
- It never removes edges. Its own note says so: "Cascade deletions for attached unmapped edges isn't strictly required".

One read of an identity or `AGENT:*` hub caches thousands of messages and edges for the life of the process, and newly projected messages and memories join them. The heap grows with what the process has touched and written, until V8's ceiling. #563 assumed this LRU evicts ("the LRU … then evicts, so the next call misses again"). It does not, because nothing sets the bound.

## The Architectural Reality

- `ai/graph/Database.mjs`: `getAdjacentNodes` with `vicinityLoadedNodes`, `lastAccessMap` and `syncCache`; `runGarbageCollector`, which turns `autoSave` and `isExecutingTransaction` off so that removing from the cache never deletes from storage; and the `nodes` and `edges` stores with their `source` and `target` indexes.
- `ai/graph/storage/SQLite.mjs#loadNodeVicinitySync` (`:938`).
- `ai/services/memory-core/GraphService.mjs:379` creates the `Database` for Memory Core. A bound belongs there as a config leaf, under ADR-0019.
- Readers that use `db.nodes.get` or `db.edges` directly must fall back to storage on a miss before eviction is safe. One already does: `MailboxService`'s projection checks. Every other such reader needs auditing.

## The Fix

1. Track every node the cache holds, including vicinity neighbors and added nodes, by last access. Past the bound, evict the least-recent nodes together with their incident edges, and drop their `vicinityLoadedNodes` flags so a later read reloads them. Eviction never deletes from storage.
2. Give Memory Core's `Database` a bound through a config leaf, per ADR-0019, sized from #1929's measurements.
3. Audit direct cache readers and give each one a storage fallback on a miss.

## Acceptance Criteria

- [ ] A unit spec (red on `dev`, where the cache only grows): vicinity loads past the bound evict the least-recent nodes and their edges, and an evicted node read again reloads from storage with identical content.
- [ ] A unit spec: eviction deletes no storage row.
- [ ] Every direct `db.nodes` / `db.edges` reader in Memory Core is listed in the PR with its fallback on a miss.
- [ ] Post-merge, on a plane cut to the merge commit: 24 h with the heap's old generation flat after warm-up, under the bound's budget, and no OOM or hang.

## Out of Scope

- The integrity repair's per-pass cost (#1958).
- The other retained sites in the profile: per-cycle collection proxies, the repair's WAL stats, and per-request `McpServer` instances. They are leads, not yet shown to grow, and go on #1843 if they do.
- Container memory and CPU sizing (#1929).

## Related

Parent #1843 · #1958 · #1929 · #563 (the eviction assumption)

Live latest-open + closed sweep: I checked the latest open issues and ran open and closed searches (graph cache eviction, `maxGraphNodes` LRU, vicinity lazy load memory) at 2026-11-27T17:36Z, plus my own assignments and a Memory Core query. No equivalent ticket exists; #563 is the closed hit, and the Problem section cites it. 153 open (red band), filed as a sub of #1843.

Retrieval Hint: "graph cache never evicts maxGraphNodes null vicinity edges heap OOM"

Origin Session ID: a3e89c2c-c549-46ca-aadf-1a2e17f7d2ec

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-27T17:37:05Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-27T17:37:07Z @neo-opus-ada added the `bug` label
- 2026-11-27T17:37:08Z @neo-opus-ada added the `ai` label
- 2026-11-27T17:37:08Z @neo-opus-ada added the `performance` label
- 2026-11-27T17:37:13Z @neo-opus-ada added parent issue #1843
### @neo-opus-ada - 2026-11-27T17:44:19Z

**Progress, and an audit finding that changes the bound's shape.** Branch `ada/1959-bounded-graph-cache`, commit `81438a6d`.

**Done: the `Database` collector.**
- Every node that enters the cache is now tracked, through the nodes Store's `mutate` event.
- Past `maxGraphNodes`, the collector evicts the least recent nodes down to 80 % of the bound, together with their incident edges.
- It clears both ends' vicinity marks, never writes to storage, and waits while a transaction runs.
- The anchor's touch and the collector now run only after `getAdjacentNodes` has built its answer.
- Three new arms in `Database.spec.mjs`. The two bounded arms ("a bounded cache evicts…" and "eviction deletes no storage row") fail on `dev`'s `Database.mjs`; the unbounded arm passes on both. The spec passes 34/34 at head.
- `maxGraphNodes: null` keeps today's behavior, so nothing changes until a consumer sets a bound.

**Finding: some Memory Core readers treat the cache as the population.** A cold-cache argument covers readers that only read lazily: each one already has to handle a miss after boot. These readers are different:
- `WakeSubscriptionService` scans `db.nodes.items` for `label === 'WAKE_SUBSCRIPTION'` (`:510`, `:1965`, `:3031`), and reads subscription nodes directly (`:1680`, `:2509`, `:2610`).
- `PermissionService#hasPermission` and `#revokePermission` scan `db.edges.items` for permission-scope edges (`:104`, `:206`). `:163` reads them from SQL instead.

Evicting a subscription node or a permission edge, or an identity node, which takes its permission edges along, would make those reads silently miss. So Memory Core's bound has to ship with pins: `WAKE_SUBSCRIPTION` nodes, permission-scope edges, and those edges' endpoints are never evicted. The remaining direct readers in `GraphService` (33 sites) and `MailboxService` (25) need the same population-versus-lazy audit before the bound goes on.

**Plan update.**
1. Add `Database` eviction pins, by node label and by edge type.
2. Have Memory Core's `GraphService` set the bound (`memoryService.graphCacheMaxNodes`, default 60,000, `NEO_MC_GRAPH_CACHE_MAX_NODES`) together with the pins.
3. Audit the 58 remaining readers, listed in the PR.

The `hasPermission` scan is also a CPU cost of its own: it walks all ~500k cached edges on every call.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-27T17:56:26Z @neo-opus-ada referenced in commit `c747dcc` - "feat(memory-core): bound the graph cache, pin what services scan (#1959)

Memory Core's Database now carries a bound from memoryService.graphCacheMaxNodes (60,000, NEO_MC_GRAPH_CACHE_MAX_NODES) and pins the populations other services read straight from the cache: WAKE_SUBSCRIPTION nodes and PermissionService's grant edges with their endpoints. Eviction runs on the macrotask after a request, so no caller that loads and then iterates in one synchronous flow sees it."
- 2026-11-27T17:57:19Z @neo-opus-ada cross-referenced by PR #1960
- 2026-11-27T19:49:54Z @tobiu referenced in commit `442f2d3` - "fix(graph): bound the Memory Core graph cache (#1959) (#1960)

* fix(graph): evict least-recent cache nodes with their edges (#1959)

The cache tracked only vicinity anchors in its LRU, kept edges forever, and ran no collector while maxGraphNodes was null, so a long-lived process held every node and edge it touched. Every node entering the cache is now tracked through the nodes Store's mutate event; past the bound the collector evicts the least recent down to 80 %, removes their incident edges, clears both ends' vicinity marks, never writes to storage, and waits while a transaction runs.

* feat(memory-core): bound the graph cache, pin what services scan (#1959)

Memory Core's Database now carries a bound from memoryService.graphCacheMaxNodes (60,000, NEO_MC_GRAPH_CACHE_MAX_NODES) and pins the populations other services read straight from the cache: WAKE_SUBSCRIPTION nodes and PermissionService's grant edges with their endpoints. Eviction runs on the macrotask after a request, so no caller that loads and then iterates in one synchronous flow sees it."
- 2026-11-27T19:49:56Z @tobiu closed this issue
### @neo-opus-ada - 2026-11-27T20:57:02Z

**Post-cut regression: the bound thrashes on message projection.** The plane was cut to `00762332` (contains #1960) at 20:49:24Z.

The bound is live: `maxGraphNodes` reads 60000. At 20:52:08Z the cache held 48,024 nodes and 37,090 edges, which is the 80 % post-eviction floor (48,000). Since then, mc-server's event loop blocks for 70–85 s at a time: 81.2 s ending 20:51:20Z, 85.3 s ending 20:52:28Z, 70.2 s ending 20:53:21Z, 83.0 s ending 20:55:17Z. The CPU sits at 96 % of its one-core quota.

In `mc_event_loop_block`, the 85.3 s block had a single job in flight, `job:message-graph-projection:2`, and the 70 s and 83 s blocks carry `message-graph-projection` too. The mechanism reads off the code. An eviction clears the vicinity mark at both ends of every evicted edge. A projection's identity hubs then reload their whole vicinities, which pushes the cache back over the bound, and the next collection evicts them again. The old code (`46228e03`) had blocks of 7–21 s in the same post-boot window.

Mitigation, pending the operator's approval (the auto-mode classifier refused it): set the live `GraphService.db.maxGraphNodes` to `null` through the container-local inspector. A restart restores the configured bound. A durable fix gets its own ticket.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-27T21:03:13Z @neo-opus-ada cross-referenced by #1964
- 2026-11-27T21:05:53Z @neo-opus-ada cross-referenced by PR #1965

