---
id: 957
title: 'A focus section in the Sandman handoff: v1 rows, views, reach, the word'
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees: []
createdAt: '2026-10-09T21:29:21Z'
updatedAt: '2026-10-09T21:29:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/957'
author: neo-fable-clio
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
blocking:
  - '[ ] 651 The Fleet Manager shows the handoff''s Focus (v1) section'
---
# A focus section in the Sandman handoff: v1 rows, views, reach, the word

## Context

Graduated from [D#19493](https://github.com/orgs/neomjs/discussions/19493) OQ-8 (the overview as a capability; the producer/reader/freshness contract from Sophie's §5.2 sweep and Euclid's input-delivery clause), as #123 Leaf 2 narrowed. The operator, 2026-10-09: peers should answer *where are we for v1, what is missing* and *which views still need love* at the start of a session; fresh sessions must check what matters cheaply. The morning surface exists — the Sandman handoff — and carries graph gaps, consolidation gaps and the computed Golden Path, nothing about v1, views or reach.

## The Problem

No artifact answers the two questions. The ROADMAP's `Row state:` lines answer the rows one by one on GitHub; the modules, the views and reach have no line anywhere; the Fleet's Golden Path pane renders only the handoff's Computed Golden Path section, because `createFleetGoldenPathSource` (`ai/services/fleet/fleetGoldenPathSource.mjs:172`) extracts exactly `## Computed Golden Path (Strategic Recommendation)` up to the next level-two heading (`:101–120`) and nothing else; the lane-landscape read is degraded on the cloud plane (the open-work census needs GitHub through the host edge).

## The Architectural Reality

- The producer: `GoldenPathSynthesizer` appends the Computed Golden Path section (`ai/services/graph/GoldenPathSynthesizer.mjs:1467`) and delegates the Consolidation Gaps section (`:948`, `frontierConsolidation.mjs`) and the routing section (`:787`, `computedGoldenPathRouting.mjs`); sections are the producer's unit.
- The readers: the Memory Core tool `get_sandman_handoff` (the file with a freshness envelope), the Fleet source above (section-addressable, with the degraded state `handoff-section-not-found`), the Fleet Manager's Golden Path pane (renders the envelope).
- #123's boundary (graduated from D#14430): mechanism public, goals and targets private, metric categories public; every `METRIC` carries `{claimClass, falsifyingQuery, windowSemantics, confoundDisclaimer, publicFlag}`; Leaf 1 delivered the schema and a git-log probe (`ai/scripts/maintenance/probeBusinessMetrics.mjs`). #123 Leaf 2 is sequenced after #122; this leaf is reporting, never a ranking gate (ADR 0033's additive boundary).
- The inputs: `Row state:` lines on the Institution's FM v1 epics (GitHub), the view receipts on Institution #505 (GitHub), GitHub traffic/stars/forks (the traffic API, push access), an outbound and blog ledger (the business repository's private workflow, hand-kept until D#19500's analytics). GitHub inputs reach the cloud producer only through an authorized host-edge reader or an admitted dated projection; no GitHub credential enters the cloud plane (Euclid, D#19493 18842685).

## The Fix

1. **The section.** A named, versioned level-two section, `## Focus (v1)`, produced by a new synthesizer section module beside `frontierConsolidation.mjs`, with four blocks: *v1* (the five rows' `Row state:` lines copied raw, never promoted — `ready` is not `passed`; the module lines when the ROADMAP carries them), *views* (one line per view key from #505's receipts: the last sweep's date and seat, the "needs love" line, open design leaves), *reach and outbound* (traffic, stars, forks with deltas; per channel the last post, last interaction, replies waiting, overdue against the cadence; the blog's last publish date against its weekly target — from the private ledger while it is hand-kept), *what waits for the operator's word* (merges and decisions, counted and linked). Every block carries its observation time, the candidate or profile it was read against, and a source link; every number its falsifying query.
2. **The inputs.** A host-edge reader supplies the GitHub-sourced blocks (the ROADMAP lines, the #505 receipts, traffic); where the host edge is absent the block reads `unknown` with the reason, as the lane-landscape read does. The private ledger is read from the business repository's workflow and projected under #123's `publicFlag` rule: categories and counts, never goals or targets. A negative public-projection test proves nothing private reaches the section.
3. **The readers.** `fleetGoldenPathSource` gains a second section-addressable read for `## Focus (v1)` with its own `handoff-section-not-found` state; the Fleet envelope carries the section as its own field (the pane leaf under Institution #312 renders it later; the pane is after v1). The `/overview` skill (neo-agent-skills) prints the same section through `get_sandman_handoff` and names it absent when the producer has not run.
4. **Stable keys.** Views by the pane's dock item id (`fleet`, `stream`, `memories`, `operator`, `tasks`, `catchUp`, `goldenPath`, `detail`) and the route views by route (`home`, `observatory`, `system`, `accounts`, `setup`, `chat`); seats by Fleet agent id.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `## Focus (v1)` section of the handoff | the synthesizer (producer-owned, versioned by a `focus.v1` marker line) | four blocks as above, each with observation time, candidate/profile, source link; raw `Row state:` words | a missing input → the block reads `unknown` + reason; the section is written even when every block is unknown | the synthesizer's section JSDoc; the handoff's reader docs | unit arm per block with fixture inputs; a snapshot arm on the section's shape |
| Host-edge reader for GitHub inputs | Euclid's clause (D#19493 18842685); the existing host-edge census pattern | reads ROADMAP lines, #505 receipts and traffic with the host's credential; hands dated rows to the producer; no credential crosses into the cloud plane | host edge absent → rows absent → `unknown` | the reader's JSDoc | unit arm: the producer runs without the reader and writes `unknown` |
| `fleetGoldenPathSource` second read | `fleetGoldenPathSource.mjs:101–120` pattern | `## Focus (v1)` extracted up to the next level-two heading; degraded state `handoff-section-not-found` | an old handoff without the section → the envelope field absent, the pane shows nothing new | the source's JSDoc | unit arm with and without the section |
| The Fleet envelope's `focus` field | the Fleet wire contract | carries the section verbatim with its freshness | — | the wire docs | the dispatch spec |
| Private → public projection | #123's `publicFlag` rule | categories, counts, dates; never goals or targets | a row without `publicFlag: true` never leaves the private workflow | #123 | a negative projection test with a seeded private row |

Decision Record impact: aligned with ADR 0033 (reporting, additive); #123's ADR-0019 (config as `AiConfig` refs) and ADR-0024 obligations bind this leaf's configuration and any persisted metric rows.

## Acceptance Criteria

- [ ] AC-1 The synthesizer writes `## Focus (v1)` with its four blocks; with every input absent the section still appears with `unknown` and reasons (unit arm).
- [ ] AC-2 `Row state:` words are copied raw; a fixture epic reading `ready` renders `ready`, never `passed` (unit arm).
- [ ] AC-3 The negative public-projection test: a seeded private ledger row without `publicFlag` never appears in the section.
- [ ] AC-4 `fleetGoldenPathSource` returns the section when present and `handoff-section-not-found` when absent; the envelope carries the `focus` field (unit arms; the dispatch spec).
- [ ] AC-5 The host-edge reader supplies the three GitHub blocks on the host plane; on the cloud plane without it the blocks read `unknown` with the reason (the lane-landscape precedent).
- [ ] AC-6 (post-merge) `get_sandman_handoff` from a seat shows the section after one synthesizer run; the `/overview` skill prints it; the operator's two questions are answered from it in one read (receipt on this leaf).

## Out of Scope

The Fleet Manager pane that renders the section (Institution #312's leaf, after v1); the social service's analytics replacing the hand-kept outbound ledger (D#19500, #123 Leaf 3); any ranking change to the Golden Path (#122); a hand-kept `VIEWS.md` (withdrawn in OQ-8: the view ledger is a generated projection from #505's receipts).

## Avoided Traps

- A second status authority: the section reports steward-owned words raw; it never promotes a row.
- Credentials into the cloud: GitHub is read on the host edge or from an admitted dated projection only.
- A section that only exists when everything is observable: every block has an `unknown` shape.
- The 256 KiB reader cap as a reading size: the section is bounded (counts, as-of, links).

## Related

Parent #123 (Leaf 2; sequenced after #122); #158 (the visibility-gap signal, ADR 0033); D#19493 OQ-8 and OQ-4 (the receipts that feed the views block); D#19500 (the outbound analytics that replace the hand ledger); D#19394 (the weekly beat's design axis); Institution #505 (the receipts), #312 (the pane leaf, after v1); neo-agent-skills (the `/overview` skill).

Live latest-open sweep: the latest 20 open issues, created-descending, at 2026-10-09 21:2xZ; no equivalent (#697 is the cloud-placement bundle, a different question). A2A in-flight claim sweep: none. Memory Core rationale: D#19493 OQ-8 and its STEP_BACK are the record.

Origin Session ID: cf93d406-6f17-4f10-9f72-9768482edfb1

Retrieval Hint: "Focus (v1) handoff section synthesizer fleetGoldenPathSource second read host-edge reader view ledger projection"

## Timeline

- 2026-10-09T21:29:23Z @neo-fable-clio added the `enhancement` label
- 2026-10-09T21:29:23Z @neo-fable-clio added the `ai` label
- 2026-10-09T21:29:23Z @neo-fable-clio added the `architecture` label
- 2026-10-09T21:29:23Z @neo-fable-clio added the `agent-os` label
- 2026-10-09T21:32:59Z @neo-fable-clio added parent issue #123
- 2026-10-09T22:00:00Z @neo-fable-clio cross-referenced by #152
- 2026-10-09T22:01:40Z @neo-gpt-sophie cross-referenced by PR #151
- 2026-10-09T22:07:37Z @neo-opus-vega cross-referenced by #651
- 2026-10-09T22:07:55Z @neo-opus-vega marked this issue as blocking #651
- 2026-10-09T23:10:48Z @neo-fable-clio cross-referenced by PR #153

