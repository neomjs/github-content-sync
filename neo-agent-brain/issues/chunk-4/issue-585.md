---
id: 585
title: A plane-attached Fleet reads PR/lane activity from the plane
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-27T14:52:43Z'
updatedAt: '2026-09-27T16:10:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/585'
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
blockedBy: []
blocking: []
closedAt: '2026-09-27T16:10:35Z'
---
# A plane-attached Fleet reads PR/lane activity from the plane

## Context

The installed Fleet Manager attaches its bundled Fleet to the local plane and reads A2A, presence and, since #582, the deployment snapshot through the admitted plane client. Its PR/lane feed still reads the local filesystem: `devFleetServer.mjs:308` and `:321` hand `AiConfig.fleet.contentRoot` to `wireFleetActivityReadSource` in both branches, including the plane one. A packaged shell carries no corpus, so the PR/lane slot degrades with `pr-lane: neo: ENOENT … scandir`. Emmy's installed receipts on neomjs/neo-agent-institution#7 (2026-09-27 11:28Z and 12:21Z) record it. Since Institution `#265`, the feed keeps the real A2A rows and labels itself partial.

Measured on `neo-local-agent-os` at Brain `742de62`, 2026-09-27 14:4xZ:
- The plane's only corpus is the orchestrator's `core-corpus-materialized/`, inside the `orchestrator-state` volume. It holds neo only. Its `_index.json`, written 14:27:29Z, lists 12,201 issues (archive included), 6,503 pulls and 307 discussions.
- mc-server and fleet-server leave `NEO_FLEET_CONTENT_ROOT` unset, have no `/app/resources/content`, and do not mount `orchestrator-state`. The only volume they share with the orchestrator is `shared-deployment-state-data`, which is how #582's snapshot reaches the Fleet.
- `get_community_activity` answers zero sources and zero items on this plane. `explore_lane_landscape` and `explore_pull_request_history` are synthesized Bird Views, not record reads.

Live latest-open sweep: checked the latest 20 open Brain issues at 14:49Z; no equivalent. A2A in-flight sweep, last 30 messages at 14:49Z: no overlapping claim; #583 (`get_graph_scene`) is the graph sibling. Memory Core sweep: Emmy's 2026-09-26 analysis and my #459 comments (5849667005, 5850818028) both name the plane-owned producer as missing and unowned. Own-assignment sweep: none of my open tickets covers it. The Tier 2.5 fork went to @neo-opus-grace, who owns the #459 boundary, at 14:41Z.

## The Problem

The #459 cleanup, readers refusing by name instead of reading an absent tree, turns the ENOENT into an honest verdict, and the feed still stays partial. A plane-attached Fleet has no plane read for PR/lane facts at all. Copying a corpus into the bundle is rejected: a remote client never has one, and the corpus belongs to the plane.

## The Architectural Reality

- The PR/lane slot (`wireFleetActivityReadSource.mjs`, `makeReadPrLaneSnapshot`) reads per origin with `resolveContentOrigins`, `readSyncedPullRecords` and `readWorkGraphIssueRecords`. It infers stall findings with `buildWorkGraphStallFindings({issuesDir, prs, now, graphService})` for the Graph origin (`CORPUS_PROJECTION_ORIGIN`) only, then hands everything to the pure `createFleetPrLaneActivitySnapshot`. The stall findings need GraphService, which is mc-server's.
- `orchestrator.corpusProjection.materializedRoot` is a formula: `materializedRootOverride ?? path.resolve(orchestrator.dataDir, 'core-corpus-materialized')` (`ai/configBase.mjs:2700`). A read-only mount at the path the leaf already resolves to in mc-server needs no new leaf and no env binding (ADR-0019 §5).
- The shape precedent is #581 / #582. The plane serves a verdict over MCP (`get_deployment_state_snapshot` → `readDeploymentInspection`, `ai/mcp/server/memory-core/toolService.mjs:573`). The Fleet reads it with `createPlaneDeploymentStateReader(planeClient)` and keeps the projection. #583's `get_graph_scene` is the same pattern in flight for the graph.
- The projector materializes into `<materializedRoot>.next-…` and swaps the tree in by directory rename (`ai/daemons/orchestrator/services/coreCorpusProjection.mjs:286–323`). A reader therefore needs the tree's parent directory, not a bind of the tree.
- The orchestrator's state directory is sole-owner by contract: `daemon.spec.mjs`'s #15759 arm asserts no other service reads or writes `orchestrator-state`. The corpus receipt already sits outside it, on the deployment-state bridge (`receiptPath` = `dirname(deploymentStateBridge.snapshotPath)/core-corpus-projection.json`), which the orchestrator writes and mc-server and fleet-server mount read-only.

## The Fix

1. **Plane placement:** `orchestrator.corpusProjection.materializedRoot` derives beside its receipt, at `dirname(deploymentStateBridge.snapshotPath)/core-corpus-materialized`. mc-server then reads it through the bridge mount it already has: no new mount, no env binding, and `orchestrator-state` keeps its sole owner. The rename swap stays inside one volume, and a missing root forces a full materialization on the next projection run (`coreCorpusProjection.mjs:421–424`).
2. **MC read:** add `get_pr_lane_activity({limit})` beside `get_deployment_state_snapshot` (openapi, Server allowlist, toolService). It runs the slot's existing path over the resolved `materializedRoot`: the readers, stall findings from mc-server's GraphService, then the pure `createFleetPrLaneActivitySnapshot`. It answers that bounded snapshot (`{capability, events}`, at most `limit` events) plus the corpus projection's watermark, read from the receipt beside the deployment snapshot on the shared volume.
   - **Raw records never cross the wire.** Measured in the orchestrator container at 14:53Z, neo's active tree is 2,430 issue files (39 MB) and 2,180 pulls (49 MB), and one issue read takes 399 ms. So the tree is read once per projection watermark, not once per call.
   - With an absent or unreadable tree, the tool answers the slot's `degraded` state with the reader's reason, never an empty success.
   - `makeReadPrLaneSnapshot` moves into its own module beside `fleetPrLaneActivityAdapter.mjs`, so mc-server imports the path without `FleetControlBridge`. No reader moves.
3. **Fleet plane reader:** `createPlanePrLaneActivityReader(planeClient)` beside `planeDeploymentStateReader.mjs` returns the plane's snapshot as the slot's answer. `wireFleetActivityReadSource` takes an injected PR/lane reader, as `wireDeploymentStateReadSource` takes `readImpl`. The `devFleetServer` plane branch passes it and stops handing `fleet.contentRoot` to the slot. In-process mode is unchanged.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `orchestrator.corpusProjection.materializedRoot` placement | the formula in `ai/configBase.mjs`, beside `receiptPath` | derives on the deployment-state bridge, which mc-server mounts read-only | before the first materialization, or mid-swap, the tool answers degraded with the reader's reason | the formula's comment | `config.template.spec` pins the derivation; `daemon.spec`'s sole-owner arm stays green |
| MC tool `get_pr_lane_activity` (new) | the slot's readers + builder over the materialized corpus, GraphService for stalls | the bounded `{capability, events}` snapshot (≤ `limit` events) + projection watermark; the tree read once per watermark | `degraded` + the reader's reason | `openapi.yaml` description | unit arms: available / absent tree / limit / one read per watermark |
| `createPlanePrLaneActivityReader` (new) + the wiring's injected reader | the plane tool's verdict | the plane branch's PR/lane slot reads the plane; `fleet.contentRoot` unread there | a throwing reader degrades the slot, never the A2A slot | module JSDoc | wiring arm red on dev |

## Decision Record impact

aligned-with ADR-0019 (§5 read at the use site; §10.5 / §10.7 the member stays with its owner, and the matrix row gains a read-only consumer mount) · aligned-with ADR 0004 (records stay origin-qualified).

## Acceptance Criteria

- [ ] **AC-1:** `materializedRoot` resolves beside its receipt on the deployment-state bridge in both the orchestrator and mc-server, with no new mount, leaf or env binding, and `orchestrator-state` keeps its sole owner. After the cut the projector rebuilds the tree there, and `get_pr_lane_activity` answers `wired` from it, including after the next swap.
- [ ] **AC-2:** `get_pr_lane_activity` answers neo's bounded PR/lane snapshot from that tree: at most `limit` events across PR, issue, lane-claim and stall rows, with the projection watermark. Two calls under an unchanged watermark read the tree once. An absent tree answers `degraded` with the reader's reason. Unit arms cover each.
- [ ] **AC-3:** in plane mode, the bundled Fleet's PR/lane slot reads the tool through the admitted client and never `fleet.contentRoot`; a throwing plane reader degrades only that slot. The wiring arm is red on dev; in-process arms stay green.
- [ ] **AC-4 (post-merge, installed — neomjs/neo-agent-institution#7):** after the plane cut and the repackage, the installed FM's activity feed reads complete rather than partial, with PR/lane rows for neo, and the shell bundle contains no corpus.

## Out of Scope

- Brain and Institution PRs: the materialization is neo-only, so a multi-origin corpus comes first (ADR 0004 §3.2.1 ids).
- #459's reader fallbacks and KB typing (@neo-opus-grace's ticket).
- The composed `fleetServer.mjs` policy (`fleetActivity: awaiting-s3`).
- Any bundle copy of the corpus.

## Avoided Traps

- **An orchestrator-written PR/lane snapshot beside the deployment state:** a second freshness clock for a tree the orchestrator already materializes.
- **Routing FM to the plane's fleet-server:** ingress routes only `/kb/mcp` and `/mc/mcp`, and that server's policy degrades `fleetActivity`.
- **Re-deriving PR/lane rows from graph nodes:** reimplements readers that already exist.
- **Mounting `orchestrator-state` read-only into mc-server (#588's first head):** breaks #15759's sole-owner contract, and hands mc-server the orchestrator's PID, lease and state files at its own resolved `orchestrator.dataDir`.
- **A `subpath` mount of the tree alone:** a bind keeps the inode that the projector's first swap renames away, so it serves a stale, then deleted, tree. Docker also refuses to start a container whose subpath does not exist yet, which bricks mc-server on a fresh plane.
- **A dedicated volume at `materializedRoot`:** the projector's rename swap cannot move a mount point.

## Related

#581 / #582 (the shape) · #583 (`get_graph_scene`, sibling) · #459 (reader fallbacks) · #442 · neomjs/neo-agent-institution#7 · neomjs/neo-agent-institution#10 (FM outcome: connected to the local Agent OS)

Retrieval Hint: `query_raw_memories("installed Fleet Manager PR/lane feed partial pr-lane ENOENT plane corpus mount mc-server")`

Origin Session ID: ea25694a-1ea9-4ba0-a357-932c7d74133a


## Timeline

- 2026-09-27T14:52:44Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-27T14:52:45Z @neo-opus-vega added the `bug` label
- 2026-09-27T14:52:46Z @neo-opus-vega added the `ai` label
- 2026-09-27T14:52:46Z @neo-opus-vega added the `agent-os` label
- 2026-09-27T15:09:54Z @neo-opus-vega cross-referenced by PR #588
- 2026-09-27T15:40:22Z @neo-opus-vega referenced in commit `479e781` - "test(fleet): the content-root census counts one wiring site — the plane branch reads the plane (#585)"
- 2026-09-27T15:52:13Z @neo-opus-vega referenced in commit `8241eff` - "fix(fleet): the materialized corpus lives beside its receipt on the deployment-state bridge — orchestrator-state stays sole-owner (#585)

The first head mounted orchestrator-state read-only into mc-server. That breaks the #15759
sole-owner contract (daemon.spec: no other service reads or writes the orchestrator's state
volume), and it would have handed mc-server the orchestrator's PID, lease and state files at
its own resolved orchestrator.dataDir.

orchestrator.corpusProjection.materializedRoot now derives beside its receipt, at
dirname(deploymentStateBridge.snapshotPath)/core-corpus-materialized. The orchestrator writes
that bridge; mc-server and fleet-server already mount it read-only. No new mount or env, and
the projector's rename swap stays inside one volume. A missing root forces a full
materialization on the next projection run, so the first cut rebuilds the tree there."
- 2026-09-27T16:10:35Z @tobiu referenced in commit `5a74360` - "Merge pull request #588 from neomjs/vega/585-plane-pr-lane-activity

fix(fleet): a plane-attached Fleet reads PR/lane activity from the plane (#585)"
- 2026-09-27T16:10:35Z @tobiu closed this issue
- 2026-09-27T16:15:23Z @neo-opus-vega cross-referenced by #64

