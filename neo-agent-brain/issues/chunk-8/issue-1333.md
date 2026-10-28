---
id: 1333
title: Deliver Focus (v1) host-edge dated projection to the container plane
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-28T04:14:59Z'
updatedAt: '2026-10-28T05:01:41Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1333'
author: neo-gemini-maryam
commentsCount: 0
parentIssue: 123
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
# Deliver Focus (v1) host-edge dated projection to the container plane

## Context

Split from #957 (`PR #1332` Round 1 & Round 2 review `RA-1 (b)` by `@neo-opus-vega`, `PRR_kwDOUBzDFM8AAAABUmU9LQ` / `PRR_kwDOUBzDFM8AAAABUmn2qg`) under parent Epic #123.

`PR #1332` lands the `## Focus (v1)` section builder (`ai/services/graph/focusV1Section.mjs`), raw `Row state:` preservation (`AC-1`, `AC-2`), `publicFlag` allowlist business-row projection (`AC-3`), `fleetGoldenPathSource` second read and `focus` envelope field (`AC-4`), the cloud-plane refusing reader (`AC-5` cloud-plane half), and the injectable host-edge reader + `gh` output parsers (`createFocusV1HostEdgeReader`, `createDefaultFocusV1HostEdgeReaders`, `parseRoadmapEpicNumbers`, `parseIssueCommentReceipts`, `parseTrafficViewsPayload`).

This leaf delivers `AC-5`'s host-plane half and `AC-6`: running the GitHub block readers in the `host-edge` process (which holds the host's GitHub credential), writing an admitted dated projection that crosses into the `container-plane` without placing a GitHub credential inside the container plane (`D#19493` OQ-8, Euclid's input-delivery clause), and wiring the in-plane reader in `GoldenPathSynthesizer` / `pipeline.mjs` to consume that admitted projection.

## The Problem

In the split `host-edge` + `container-plane` Agent OS topology (`ADR 0014`, `ai/daemons/orchestrator/taskAuthority.mjs`, `deploy/cloud/docker-compose.local-agent-os.yml`), `GoldenPathSynthesizer` (`golden-path` task, `ai/daemons/orchestrator/scheduling/pipeline.mjs:706`) and `MemoryService#mutateFrontier` (`ai/services/memory-core/MemoryService.mjs:3317`) execute inside the `container-plane` (`orchestrator` and `mc-server` containers).

Inside the `container-plane` containers:
1. `command -v gh` finds no binary and no `GH_*` / `GITHUB_*` credential exists.
2. Per Euclid's input-delivery clause (`D#19493` `18842685`), no GitHub credential may be placed inside the cloud/container plane.
3. Spawning `gh` directly inside `pipeline.mjs` or `MemoryService.mjs` therefore fails on the container plane with `ENOENT` instead of reading host-edge GitHub data.

To populate the three GitHub-sourced `## Focus (v1)` blocks (`v1 — where it stands`, `Views — design sweep (#505)`, `Reach & outbound`) and the GitHub half of `Awaiting the operator's word` on a host-paired deployment, the `gh` block readers in `focusV1Section.mjs` must run on the `host-edge` (where the host credential lives) and hand a dated, admitted projection to the `container-plane` producer.

## The Architectural Reality

Design authority:
- `D#19493` OQ-8 and Euclid's input-delivery clause (`18842685`): GitHub inputs for `## Focus (v1)` are read on the host edge or from an admitted dated projection; no GitHub credential enters the cloud/container plane.
- `learn/agentos/decisions/0014-cloud-deployment-topology-and-scheduler-task-taxonomy.md` (`ADR 0014`) & `ai/daemons/orchestrator/taskAuthority.mjs`: `golden-path` is a `container-plane` task (`taskAuthority.mjs:114`), whereas host-credentialed and host-loopback effects run under `ORCHESTRATOR_AUTHORITY_CLASS.hostEdge` (`src/composition/orchestrator/hostEdgeProfile.mjs`).
- `learn/agentos/decisions/0019-aiconfig-reactive-provider-ssot.md` (`ADR 0019`): any new path or admission coordinate must be declared as a reactive `AiConfig` leaf in `ai/configBase.mjs` with parity in `ai/scripts/lint/config-leaf-parity.json` (following the `wakeReceiverRecordsDir` read-only bind pattern at `ai/configBase.mjs:535` / `deploy/cloud/docker-compose.local-agent-os.yml:84-99` or the `admit_community_batch` MCP admission pattern).
- Structure Map (`node ./ai/scripts/diagnostics/structureMap.mjs --files --loc`):
  - `ai/services/graph` (`41` files): owns `focusV1Section.mjs` and `GoldenPathSynthesizer.mjs`.
  - `ai/daemons/orchestrator` (`6` files) & `ai/daemons/orchestrator/scheduling` (`19` files): own `hostEdge.mjs`, `taskAuthority.mjs`, `registry.mjs`, and `pipeline.mjs`.

## The Fix

1. **Host-Edge Collector Execution (`ai/services/graph/focusV1Section.mjs` + `host-edge` runner):**
   - Execute `createDefaultFocusV1HostEdgeReaders` / `createFocusV1HostEdgeReader({useDefaultHostReaders: true})` on the `host-edge` (where `gh` and the host's GitHub credentials exist) to collect dated rows for:
     - `v1Rows` (`neomjs/neo-agent-institution` `ROADMAP.md` `## Focus (v1)` epics and their raw `Row state:` lines),
     - `viewsReceipts` (`neomjs/neo-agent-institution#505` `[design-sweep]` comments and `package.json` version),
     - `reachTraffic` (`/repos/neomjs/neo/traffic/views`, stargazers, forks), and
     - `operatorWord` (`needs-operator` open issues across the 6 organism repos and approved open PRs awaiting human merge).
   - Write an atomic, schema-versioned (`focus-v1-projection.v1`), dated JSON projection containing only the public-safe block payloads and capture timestamps.
2. **Admitted Cross-Plane Delivery & In-Plane Reader (`ai/services/graph/focusV1Section.mjs`, `ai/daemons/orchestrator/scheduling/pipeline.mjs`):**
   - Expose an in-plane projection reader (`createFocusV1ProjectionReader({projectionPath})`) in `ai/services/graph/focusV1Section.mjs` that reads and validates the admitted dated projection file (bound via an `AiConfig` leaf per `ADR 0019`, or admitted into the plane data root) and feeds `createFocusV1HostEdgeReader`.
   - When the projection file is absent or unconfigured (e.g. a pure cloud-plane deployment with no host edge), the reader returns `null` per block so `renderFocusV1Section` emits `unknown` with the plane-appropriate reason (`CLOUD_PLANE_HOST_EDGE_ABSENT_REASON` when `repoEnrichmentEnabled === false`, or `DEFAULT_HOST_EDGE_ABSENT_REASON` when unconfigured).
   - Wire `pipeline.mjs:706` (`runGoldenPathTask`) and `MemoryService.mjs:3317` (`mutateFrontier`) to pass the projection-backed reader on in-plane synthesis passes, never spawning `gh` inside the container plane.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| Host-edge Focus (v1) projection writer | `D#19493` OQ-8 (`18842685`); `focusV1Section.mjs` `createDefaultFocusV1HostEdgeReaders` | Runs `gh` block readers on the host edge and writes a dated `focus-v1-projection.v1` JSON artifact | `gh` or network failure on a single block records that block's `reason` in the projection without dropping sibling blocks | Module JSDoc in `focusV1Section.mjs` | Unit test driving the host-edge writer with injected `execGh` and verifying the written projection JSON |
| In-plane projection reader (`createFocusV1ProjectionReader`) | `focusV1Section.mjs` `createFocusV1HostEdgeReader` seam; `ADR 0019` | Reads the admitted dated projection in `pipeline.mjs` / `MemoryService.mjs` without spawning `gh` or holding credentials in the plane | Missing, corrupt, or unconfigured projection path degrades each block to `unknown (<reason>)` | Module JSDoc in `focusV1Section.mjs` | Unit arms for valid projection, partial-block projection, missing projection, and `runGoldenPathTask` production path |

Decision Record impact: `aligned-with ADR 0014, ADR 0019, ADR 0033`.

## Acceptance Criteria

- [ ] AC-1 A host-edge runner executes `createDefaultFocusV1HostEdgeReaders` on the host edge and emits a dated `focus-v1-projection.v1` payload without running inside the container plane (unit arm).
- [ ] AC-2 `createFocusV1ProjectionReader` in `ai/services/graph/focusV1Section.mjs` consumes the admitted dated projection and supplies the four Focus (v1) blocks to `GoldenPathSynthesizer#synthesizeGoldenPath` without spawning `gh` in the container plane (unit arm).
- [ ] AC-3 `runGoldenPathTask` (`ai/daemons/orchestrator/scheduling/pipeline.mjs`) and `MemoryService#mutateFrontier` wire the projection reader on production synthesis runs when `repoEnrichmentEnabled !== false`, and keep `makeRefusingFocusV1Reader(CLOUD_PLANE_HOST_EDGE_ABSENT_REASON)` when `repoEnrichmentEnabled === false` (unit arm driving the production call path).
- [ ] AC-4 Any new configuration coordinate follows `ADR 0019` (`ai/configBase.mjs` leaf + `ai/scripts/lint/config-leaf-parity.json`) and passes `lint-script-plane.mjs`.
- [ ] AC-5 Complete #957 `AC-5` (host-plane half) and `AC-6` (`get_sandman_handoff` from a seat shows the populated section after one synthesizer run; `/overview` prints it).

## Out of Scope

- Changing the `## Focus (v1)` Markdown layout, 2,000-byte budget, or `publicFlag` allowlist rules already shipped in `PR #1332`.
- Changing the `fleetGoldenPathSource` second read or `focus` wire envelope field already shipped in `PR #1332`.
- The Fleet Manager UI pane (`neomjs/neo-agent-institution#312`).

## Avoided Traps

- **Spawning `gh` inside the container plane:** `orchestrator` and `mc-server` run in containers without `gh` or GitHub credentials. All `gh` execution stays on the host edge.
- **Putting GitHub credentials into the container plane:** Violates Euclid's clause (`D#19493` `18842685`). Only the dated, public-safe projection crosses into the plane.
- **Ad-hoc `process.env` reads in service or pipeline modules:** Violates `ADR 0019` §3.

## Related

- Parent Epic: #123
- Sibling / Prerequisite PR: #1332 (lands `focusV1Section.mjs` builder, parsers, `createFocusV1HostEdgeReader` seam, and `fleetGoldenPathSource` `focus` axis)
- Target outcome ticket kept open until this leaf lands: #957 (`AC-5` host-plane half and `AC-6`)

unowned-rationale: Sequenced immediately after PR #1332 merges the `focusV1Section.mjs` reader and parser seam; open for pickup once #1332 lands on `dev`.

Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-brain` (`#1299` down to `#850`) at `2026-10-28T04:13:19Z`; no duplicate found (`#957` is the parent leaf whose host-plane projection wiring is split here per `PR #1332` `RA-1 (b)`).
A2A in-flight claim sweep: checked `list_messages({status: 'all', limit: 30})` at `2026-10-28T04:13:25Z`; no competing claim on Focus (v1) host-edge projection delivery.
Memory Core rationale sweep: `query_raw_memories({query: 'host-edge Focus v1 dated projection container-plane orchestrator gh credential'})` at `2026-10-28T04:13:33Z`; confirmed `PR #1332` Round 1 & Round 2 review context.
Own-assignment sweep: checked `gh issue list -R neomjs/neo-agent-brain --state open --assignee @me` at `2026-10-28T04:13:19Z` (`#957` only).

Origin Session ID: 58b07b45-1fe9-40ad-80f8-db0f380332b8

Retrieval Hint: "Focus (v1) host-edge dated projection createFocusV1ProjectionReader createDefaultFocusV1HostEdgeReaders #957 #123"


## Timeline

- 2026-10-28T04:15:01Z @neo-gemini-maryam added the `enhancement` label
- 2026-10-28T04:15:01Z @neo-gemini-maryam added the `ai` label
- 2026-10-28T04:15:01Z @neo-gemini-maryam added the `architecture` label
- 2026-10-28T04:15:02Z @neo-gemini-maryam added the `agent-os` label
- 2026-10-28T04:15:24Z @neo-gemini-maryam added parent issue #123
- 2026-10-28T04:15:25Z @neo-gemini-maryam cross-referenced by #957
- 2026-10-28T04:19:15Z @neo-gemini-maryam cross-referenced by PR #1332
- 2026-10-28T05:01:41Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-28T05:18:00Z @neo-gemini-maryam cross-referenced by PR #1337
- 2026-10-28T05:27:03Z @neo-gemini-maryam referenced in commit `3e1019f` - "fix(ai): guard focusV1ProjectionPath formula when handoffFilePath is unset (#1333)"
- 2026-10-28T06:26:44Z @neo-gemini-maryam referenced in commit `6a2b796` - "fix(ai): bind host-edge Focus v1 projection into local container plane (#1333)"
- 2026-10-28T06:35:08Z @neo-gemini-maryam referenced in commit `b60cb3e` - "style(ai): remove decay-prone comment refs in hostEdge and volume scoping spec (#1333)"

