---
id: 1251
title: get_sandman_handoff answers the Golden Path only; the gap sections by request
state: OPEN
labels:
  - enhancement
  - ai
  - performance
  - agent-os
assignees: []
createdAt: '2026-10-24T20:13:07Z'
updatedAt: '2026-10-24T20:13:07Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1251'
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
# get_sandman_handoff answers the Golden Path only; the gap sections by request

## Context

The operator's direction (2026-10-24, restated after an earlier challenge): a seat's boot read of the handoff should return the computed Golden Path only; the rest of the Markdown comes back only when a parameter asks for it. Measured tonight on a Claude seat: a full context recovery reached 125k tokens before the first word of work, 89k of them fixed preload. Every boot read that returns more than the question asked compounds that across ten seats and three resets a week.

## The Problem

`get_sandman_handoff` returns the whole `sandman_handoff.md`. Today that is ~7.5 KB (≈ 2k tokens): five typed gap sections (test constraints, guide disconnects, example disconnects, orphaned concepts, reverification queue), the Consolidation Gaps section (178 undigested sessions, five shown) and, last, the `## Computed Golden Path (Strategic Recommendation)` section a seat actually boots on. When #957's `## Focus (v1)` section lands the file grows again. A seat that wants the route pays for the audit; a seat that wants the audit has no way to ask for one section. The Fleet already solved the reading side once: `fleetGoldenPathSource.mjs` extracts exactly the Golden Path heading up to the next level-two heading (per #957's Architectural Reality), so the plane has two readers of one file with two contracts.

## The Architectural Reality

- Producer: `ai/services/graph/GoldenPathSynthesizer.mjs` writes the gap sections as `###` blocks (`:1730–1755`), delegates Consolidation Gaps (`:1786`, `frontierConsolidation.mjs`) and appends `## Computed Golden Path (Strategic Recommendation)` (`:1503`). Sections are the producer's unit.
- Reader: `ai/services/memory-core/helpers/sandmanHandoffStore.mjs:27` `readSandmanHandoff({filePath, now, staleAfterMs, maxBytes = 256 KiB})` returns `{content, path, mtimeMs, ageMs, staleAfterMs, stale, reason}` with the whole file as `content`; `ai/mcp/server/memory-core/toolService.mjs:103` `readSandmanHandoffTool` forwards only `staleAfterMs`; the OpenAPI operation `get_sandman_handoff` sits at `ai/mcp/server/memory-core/openapi.yaml:464–` (`/handoff/sandman`).
- The route also has its own structured tool: `get_computed_route` (`computedRouteStore.mjs`, the `computed-route.v1` sidecar, ~3k tokens as JSON). The handoff's Golden Path section is the prose rendering of that sidecar.
- Design authority: the reader's JSDoc (`sandmanHandoffStore.mjs`) documents serving the whole file as the file-contract half of the tool; this ticket changes that contract on the operator's direction, it does not call the current behaviour a defect.

## The Fix

1. A section splitter beside the store (sibling-file lift; the Fleet source's heading-extraction pattern, generalised): split `content` on level-two headings into named sections — `gaps` (everything before the first `## `, i.e. the `###` audit blocks), `consolidation`, `goldenPath`, and any later producer section by its slug (`focus` once #957 lands).
2. `readSandmanHandoff` gains `sections` (array of slugs) and the tool declares it: **default `['goldenPath']`**; `['*']` or `'all'` returns the whole file as today. The envelope keeps `path, mtimeMs, ageMs, stale, reason` and adds `sections: {<slug>: {content, bytes}}` plus `availableSections` (slugs present) so a caller learns what it did not ask for without paying for it. `content` stays populated only for `all`.
3. The Fleet source keeps its own extractor (its degraded state `handoff-section-not-found` is already a contract); a follow-up may point it at the splitter, out of scope here.
4. `openapi.yaml` declares `sections` with the enum and the default; the service-parity lint (`ai/scripts/lint/lint-openapi-service-parity.mjs`) must see the parameter reach the bound function, the way `depth` on `get_context_frontier` did not (`:136–143`).

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `get_sandman_handoff` (MCP, `openapi.yaml:464–`) | the tool's x-neo-tool-summary + the store's JSDoc | `sections` param, default `['goldenPath']`; `all` = today's payload | unknown slug → `sections[slug] = null` + listed in `missingSections`; file missing → today's `content: null` envelope unchanged | openapi description + store JSDoc | unit arms: default returns only the route section; `all` is byte-identical to today; unknown slug |
| `readSandmanHandoff` (`sandmanHandoffStore.mjs:27`) | its JSDoc | same param; the splitter is pure and exported for the Fleet source to adopt later | a file without the Golden Path heading → `goldenPath: null`, `availableSections` lists what exists | JSDoc | fixture handoff with and without the heading |
| the Fleet source (`fleetGoldenPathSource.mjs`) | its own section contract | unchanged | — | — | its existing spec stays green |

Decision Record impact: none (an additive parameter; the handoff file format is untouched; ADR 0033's additive boundary holds).

## Acceptance Criteria

- [ ] AC-1 `get_sandman_handoff` with no arguments returns the `## Computed Golden Path` section only, inside the freshness envelope; the response is under 1,000 tokens on today's file (unit arm with the current fixture; the byte count asserted).
- [ ] AC-2 `sections: 'all'` returns `content` byte-identical to today's whole-file read (regression arm).
- [ ] AC-3 `sections: ['gaps','consolidation']` returns those two and lists `goldenPath` under `availableSections` without its content.
- [ ] AC-4 The OpenAPI operation declares `sections`; `npm run lint` (service parity) passes with the parameter reaching `readSandmanHandoff`.
- [ ] AC-5 (post-merge) a fresh seat's recovery that calls the tool bare logs the smaller payload; the context-recovery runbook's step 4 names the default.

## Out of Scope

Changing what the synthesizer writes; the Fleet source's extractor; #957's `## Focus` section content (it becomes one more slug here, nothing else); the computed-route sidecar.

## Avoided Traps

- A second handoff file per section: one producer, one file, one splitter; sections stay addressable by heading.
- Declaring the parameter in OpenAPI without passing it through (the `depth` precedent the parity lint dispositioned on #16611).
- Truncating by bytes instead of by section: a seat must never receive half a section.

## Related

Sibling filings tonight on the same measurement: the compact `get_context_frontier` default, the compact `list_messages` projection, the boot Focus section (sub of #123), the skills description budget (neo-agent-skills). #957 (the Focus section this reader will serve), #496 (the Fleet reads the route through one method), #16611 (the parity lint's `depth` disposition).

unowned-rationale: a one-PR leaf any seat can take; the filer's week is committed (the business block, outcomes 3, 7, 8); the measurement in Context is the premise, nothing else is pending.

Live latest-open sweep: the latest 20 open issues, created-descending, at 2026-10-24 19:5xZ; no equivalent (#957 is the Focus section, a different question). A2A in-flight claim sweep: the latest 30 `[lane-claim]` rows, none on this scope. MC sweep: `get_sandman_handoff returns graph gaps consolidation gaps golden path section payload size operator challenged`, 6 results, initialization noise, no prior decision found. Own-assignment sweep: 11 open, none overlapping. Structure map: `ai/services/memory-core/helpers` (the store), `ai/mcp/server/memory-core` (the tool), `ai/services/graph` (the producer) — owning folders exist; the splitter is a sibling-file lift beside `sandmanHandoffStore.mjs`.

Origin Session ID: 542d408d-b0eb-497c-8f57-a19ca5833f5a

Retrieval Hint: "sandman handoff sections param golden path only default boot read token cost"

## Timeline

- 2026-10-24T20:13:08Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-10-24T20:13:09Z @neo-fable-mnemosyne added the `ai` label
- 2026-10-24T20:13:09Z @neo-fable-mnemosyne added the `performance` label
- 2026-10-24T20:13:09Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-10-24T20:17:12Z @neo-fable-mnemosyne cross-referenced by #957

