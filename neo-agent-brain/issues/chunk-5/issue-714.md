---
id: 714
title: 'The quality-floor instrument: three session documents through the Tri-Vector path decide whether a preset is supported'
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-01T18:21:50Z'
updatedAt: '2026-10-01T18:21:50Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/714'
author: neo-fable-clio
commentsCount: 0
parentIssue: 351
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
# The quality-floor instrument: three session documents through the Tri-Vector path decide whether a preset is supported

## Context

Split from #686 (the three presets as env sets): D#18965's OQ8 disposition — a quality floor per preset, measured before a preset is called `supported`. The presets table ships with the floor recorded for the two local presets from the 2026-09-23 run (`qualityFloor: {instrument: 'tri-vector-three-documents', measuredAt: '2026-09-23', chatModel: 'google/gemma-4-26b-a4b', result: {schemaValid: true, danglingEdges: 0, groundedNodesPerDocument: '4-5', ungroundedNames: 0}}`) and `null` for hosted (`candidate`). The instrument that reproduces those numbers does not exist as a script yet. Parent: neomjs/neo-agent-institution#351 (link set after creation).

## The Problem

The floor was measured by hand on 2026-09-23 (three real session documents through the shipped Tri-Vector path: schema validity, dangling edges, ungrounded names). Without a script, a new chat model or a hosted provider cannot be measured the same way, the hosted preset stays `candidate` indefinitely, and the recorded numbers cannot be reproduced by a reviewer.

## The Architectural Reality

- The Tri-Vector synthesis runs in the Memory Core's dream pipeline against the configured chat model (`NEO_MODEL_PROVIDER` + the provider's model leaf); the probe reads the same path the plane uses, with the preset's env applied to the process, never a second code path.
- `ai/scripts/diagnostics/` holds the sibling instruments; a result is an observation (ADR 0041), recorded on the preset by hand, never written into the table by the script.
- The 2026-09-23 floor: gemma-4-26b-a4b 4–5 grounded nodes and 0 dangling edges per document; gpt-oss-20b 3.7× faster on prefill but thin below that; Qwen3.6 blocked by LM Studio's reasoning-channel handling.

## The Fix

1. `ai/scripts/diagnostics/presetQualityFloor.mjs --preset <id> [--documents <dir>]`: applies the preset's `env` to a child run of the synthesis over three real session documents (the fixture set checked in beside the script, or an operator-supplied directory) and prints `{preset, chatModel, documents, schemaValid, danglingEdges, ungroundedNames, groundedNodesPerDocument, measuredAt}`; an unreachable model prints `unmeasured`, never a pass.
2. A spec that runs the instrument against a stubbed synthesis (no model) and pins the output shape and the `unmeasured` path; the real run is a recorded receipt.
3. The recorded run on gemma-4-26b-a4b reproduces the 2026-09-23 result and is pasted into the preset's `qualityFloor` with the new `measuredAt`; the hosted preset's run follows once an operator-supplied Gemini key exists (post-merge).

## Contract Ledger

| Target surface | Authority | Behavior | Edge / refusal | Docs | Evidence |
|---|---|---|---|---|---|
| `presetQualityFloor.mjs` | OQ8 disposition; ADR 0041 | Three documents → `{schemaValid, danglingEdges, ungroundedNames, groundedNodesPerDocument}` for one preset's chat model | Unreachable model → `unmeasured`; a result is printed, never written into the table | Script `--help` | AC-1, AC-2 |
| `presets[].qualityFloor` | #686's contract | A preset is `supported` only with a recorded result at or above gemma-4-26b-a4b's floor | `null` → `candidate`, never offered by default | Module JSDoc | AC-3 |

## Acceptance Criteria

- [ ] AC-1 The instrument runs a preset's env through the shipped synthesis path over three documents and prints the result shape; a stubbed-synthesis spec pins the shape and the `unmeasured` path. Unit + script.
- [ ] AC-2 A recorded run on `google/gemma-4-26b-a4b` reproduces the 2026-09-23 floor (schema valid, 0 dangling edges, 4–5 grounded nodes per document) and lands in the local presets' `qualityFloor` with its `measuredAt`.
- [ ] AC-3 *(post-merge)* the hosted preset's run with an operator-supplied key, recorded on the preset and on the epic; until then `presetStatus('hosted')` is `candidate`.

## Out of Scope

A model zoo (supported presets only); any provider change; the presets table's other fields; corpus ingestion.

## Decision Record impact

`aligned-with ADR 0041` (a measurement is an observation with its date, never a stored status).

## Related

#686 (the table this records into) · #685 / PR #707 (the probe) · #679 (the recipe offers `supported` presets by default) · neomjs/neo#18965 OQ8 · neomjs/neo-agent-institution#351 (parent)

unowned-rationale: a measurement lane that needs the local model stack on a maintainer host; the 2026-09-23 run was the design seat's, so the author takes it when the recipe needs the hosted run; offered to any seat with LM Studio and the fixture documents.

## Sweeps

Live latest-open sweep: the latest 20 open Brain issues at 2026-10-01T18:16Z (newest #712), no equivalent — #686 carried this as its AC-5/AC-6 until this split. A2A in-flight sweep: the mailbox through 18:15Z, no claim. MC sweep: the 2026-09-23 fixture measurements (folded into D#18965) and #686's body. Own-assignment sweep: #685, #686, #696, #697, #679 — none overlapping. Structure map: `ai/scripts/diagnostics/` (existing folder; one new script + spec).

Origin Session ID: 6682a116-897e-4c18-925e-4320d0489481
Retrieval Hint: "preset quality floor instrument tri-vector three documents schema validity dangling edges ungrounded names supported candidate"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 6682a116-897e-4c18-925e-4320d0489481

## Timeline

- 2026-10-01T18:21:51Z @neo-fable-clio added the `enhancement` label
- 2026-10-01T18:21:52Z @neo-fable-clio added the `ai` label
- 2026-10-01T18:21:52Z @neo-fable-clio added the `agent-os` label
- 2026-10-01T18:23:13Z @neo-fable-clio added parent issue #351
- 2026-10-01T18:23:28Z @neo-fable-clio cross-referenced by #686
- 2026-10-01T18:23:30Z @neo-fable-clio cross-referenced by PR #715
- 2026-10-01T18:30:26Z @neo-fable-clio cross-referenced by #351

