---
id: 1252
title: 'get_context_frontier answers compact by default: counts and the top neighbours, the full topology by request'
state: OPEN
labels:
  - enhancement
  - ai
  - performance
  - agent-os
assignees: []
createdAt: '2026-10-24T20:13:44Z'
updatedAt: '2026-10-24T20:13:44Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1252'
author: neo-fable-mnemosyne
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
# get_context_frontier answers compact by default: counts and the top neighbours, the full topology by request

## Context

Measured 2026-10-24 19:3xZ from a Claude seat: `get_context_frontier` returned 160,030 characters (≈ 40k tokens). 149,881 of them were `topology.strategicNeighbors`: 446 nodes, of which 402 are `SESSION_SUMMARY`, 22 `AGENT_MEMORY`, 10 `ISSUE`, 9 `SESSION`, 1 `CLASS`, 2 `System`. `semanticContexts` held 4 entries (9,827 characters). The harness refused the result and wrote it to disk; nothing a seat boots on can be that size. The operator's direction (2026-10-24): the frontier read returns a tiny result, like the Golden Path; the full topology only when asked.

## The Problem

The frontier is the graph's "what matters now" anchor and the June business-on-the-graph concept named it as the surface that would put a business node in front of the swarm. At 40k tokens it cannot be a boot read for anyone, and its size grows with every digested session: every edge of weight ≥ 0.8 touching `frontier` is returned with the neighbour's `description`, which for a session summary is the summary. The four entries with a `semanticVectorId` are fetched from Chroma one by one; the other 442 are returned as topology only, so the caller pays for 442 descriptions it cannot use.

## The Architectural Reality

- `ai/services/memory-core/GraphService.mjs:1879` `getContextFrontier({depth = 2})` collects every inbound and outbound edge of `frontier` with `weight >= 0.8`, filters RLS and CLOSED/archived, pushes `{id, type, name, description, semanticVectorId, relationship, weight, direction}` per neighbour, sorts by weight — no cap, no type filter, `depth` unused by the sole caller.
- `ai/services/memory-core/MemoryService.mjs:2836–2898` `getContextFrontier()` takes no parameters, calls the GraphService method with nothing, fetches one Chroma document per neighbour that has a `semanticVectorId`, and returns `{_channelSeparation, topology, semanticContexts}` through the projection admission wrapper.
- `ai/mcp/server/memory-core/toolService.mjs:604` binds the tool; the OpenAPI operation is at `openapi.yaml:1610`; the parity lint already records that `depth` is declared but unreachable (`lint-openapi-service-parity.mjs:136–143`, dispositioned on #16611) — the pattern this ticket must not repeat.
- Design authority: the method's JSDoc describes the full topology as the intended return; the contract changes on the operator's direction, no defect is claimed.

## The Fix

1. `MemoryService.getContextFrontier({projection = 'compact', neighborLimit = 12, types} = {})` — declared on the bound method and passed through, so the parity lint sees a reachable parameter.
2. `projection: 'compact'` (default) returns: the frontier node (`id, type, name`), `neighborCounts` by `type`, `totalNeighbors`, the top `neighborLimit` neighbours by `weight` as `{id, type, name, relationship, weight, direction}` **without `description`**, and `semanticContexts` trimmed to `{nodeId, name, weightedScore, trustTier}` plus the first 300 characters of `content`. Target: under 1,500 tokens on today's graph (asserted as a byte count in the unit arm).
3. `projection: 'full'` returns today's payload unchanged. `types` filters neighbours by label in either projection (e.g. `['ISSUE','AGENT_MEMORY']` for a seat that wants work and memories, not session summaries).
4. `openapi.yaml` declares the three parameters with defaults; the tool summary says the default is compact.
5. GraphService gains the cap and the type filter as parameters with today's behaviour as the no-argument path; the `depth` disposition stays as the lint records it.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `get_context_frontier` (MCP, `openapi.yaml:1610`) | the tool summary + MemoryService JSDoc | `projection` default `compact`; `neighborLimit`; `types` | no frontier node → today's `{message}` envelope; `neighborLimit` ≤ 0 rejected like `list_messages` rejects `limit: 0` | openapi + JSDoc | unit arms: compact under the byte bound on a fixture of 450 neighbours; `full` byte-identical to today; `types` filter |
| `GraphService.getContextFrontier` (`:1879`) | its JSDoc | optional `{limit, types}`; no-argument call unchanged | — | JSDoc | existing arms green; one new arm per parameter |
| the projection admission wrapper | `buildContextFrontierProjectionFallback` | unchanged; wraps either projection | — | — | existing arms |

Decision Record impact: none (additive parameters; the graph is untouched).

## Acceptance Criteria

- [ ] AC-1 A bare `get_context_frontier` on a graph with ≥ 400 frontier neighbours returns a compact payload under 6,000 characters (unit arm with a seeded fixture; the count asserted) and carries `neighborCounts` and `totalNeighbors`.
- [ ] AC-2 `projection: 'full'` is byte-identical to today's return on the same fixture.
- [ ] AC-3 `types: ['ISSUE']` returns only ISSUE neighbours in both projections.
- [ ] AC-4 The OpenAPI operation declares the parameters; the parity lint passes with them reachable; `depth` keeps its recorded disposition.
- [ ] AC-5 (post-merge) a seat's recovery can call the tool bare without the harness spilling the result to disk (receipt: the response size in the turn).

## Out of Scope

Changing how the Dream writes frontier edges or their weights; `mutate_frontier`; the business focus node's placement (the sibling filing under #123 decides its surface; the frontier is not it at today's size).

## Avoided Traps

- A cap without counts: a reader must see how much it did not get.
- Dropping session summaries silently: they stay countable and reachable through `types` or `full`.
- Declaring parameters the bound method does not receive (the `depth` precedent).

## Related

Sibling filings tonight: the Golden-Path-only `get_sandman_handoff` default, the compact `list_messages` projection, the boot Focus section (sub of #123), the skills description budget (neo-agent-skills). #16611 (the parity lint disposition), #122 (Golden Path v2, where a ranked frontier would come from).

unowned-rationale: a one-PR leaf any seat can take; the filer's week is committed (the business block, outcomes 3, 7, 8); the measurement in Context is the premise.

Live latest-open sweep: the latest 20 open issues, created-descending, at 2026-10-24 19:5xZ; no equivalent. A2A in-flight claim sweep: the latest 30 `[lane-claim]` rows, none on this scope. MC sweep: `get_context_frontier strategicNeighbors session summaries payload too large boot read tokens`, 6 results, initialization noise, no prior decision found. Own-assignment sweep: 11 open, none overlapping. Structure map: `ai/services/memory-core` (GraphService, MemoryService), `ai/mcp/server/memory-core` (toolService, openapi) — owning folders exist; no new file.

Origin Session ID: 542d408d-b0eb-497c-8f57-a19ca5833f5a

Retrieval Hint: "context frontier compact projection neighbor counts session summaries 160k payload"

## Timeline

- 2026-10-24T20:13:46Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-10-24T20:13:46Z @neo-fable-mnemosyne added the `ai` label
- 2026-10-24T20:13:46Z @neo-fable-mnemosyne added the `performance` label
- 2026-10-24T20:13:46Z @neo-fable-mnemosyne added the `agent-os` label

