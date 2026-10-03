---
id: 798
title: The hosted preset records its quality floor and becomes supported
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-fable-clio
createdAt: '2026-10-03T07:45:01Z'
updatedAt: '2026-10-03T08:45:00Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/798'
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
closedAt: '2026-10-03T08:45:00Z'
---
# The hosted preset records its quality floor and becomes supported

The promotion leaf Grace's rule on #746 names: three consecutive met samples under the merged #777 rule put a `HOSTED_FLOOR` row beside `GEMMA_FLOOR`, and the hosted preset carries it as its `qualityFloor` — so `presetStatus(hosted)` reads `supported` and the first-run recipe may RECOMMEND it where it fits, instead of listing it as "possible, never recommended by default". Sub of neomjs/neo-agent-institution#351 (row 1: the wizard's supported profiles).

## Context

`placementPresets.mjs` records a preset's floor as the reference run of the quality-floor instrument (#714) over the shipped three-document fixture set; a preset without one is a `candidate` (`presetStatus`, `:222`). The local presets carry `GEMMA_FLOOR` (2026-10-02). The hosted preset was left `qualityFloor: null` by #686 pending measurement; #746's AC-4 ledger holds the samples: sample 1 (pre-#777 rule) read one invention, samples 2–4 (2026-10-02/03, the merged rule, `gemini-3.8-flash` through Gemini's OpenAI-compatible endpoint) read 5–7 / 4–8 / 4–5 grounded nodes per document, 0 dangling edges, 0 ungrounded names — https://github.com/neomjs/neo-agent-brain/issues/746#issuecomment-5966875849.

## The Problem

A host without a GPU (every stranger's laptop the plan's §4 names) can only run the hosted preset, and today the recipe tells that operator "nothing recommended; possible: hosted (no recorded quality floor…)". The measurement that would change the sentence exists and meets the bar; the row does not.

## The Architectural Reality

- `ai/services/fleet/placementPresets.mjs` — `GEMMA_FLOOR` (`:110–119`) is the shape: `instrument`, `measuredAt`, `chatModel`, `documents`, `documentsDigest`, `result`, `note`; the hosted preset (`:150–173`) holds `qualityFloor: null`; `presetStatus` (`:222`) is the only reader.
- `ai/services/fleet/firstRunRecipe.mjs#recommendPlacement` — a `supported` preset that fits with headroom is recommended; a `candidate` is only ever possible.
- Specs that pin the candidate status: `placementPresets.spec.mjs:95–101`, `firstRunRecipe.spec.mjs:209–244` (the headroom and placement-step arms), the CLI's AC-5 arm reads the recommendation list.

## The Fix

One constant, one field, the specs that pinned the old status:

1. `HOSTED_FLOOR = Object.freeze({instrument: 'tri-vector-three-documents', measuredAt: '2026-10-03', chatModel: GEMINI_FLASH, documents: <the fixture set>, documentsDigest: 'f3cd8b71…', result: {schemaValid: true, danglingEdges: 0, groundedNodesPerDocument: '4-5', ungroundedNames: 0}, note: <the three receipts, the pre-rule miss, the endpoint>})` — the recorded result is the WEAKEST of the three met samples (sample 4), never the best.
2. hosted `qualityFloor: HOSTED_FLOOR`.
3. Specs: `presetStatus(hosted)` → `supported`; the recipe's placement arms read hosted as recommended where it clears the headroom (the generous host) and still possible-not-recommended only by headroom, never by status; the CLI's recommendation line gains `hosted`.

## Contract Ledger

| Surface | Authority | Behavior | Edge case | Docs | Evidence |
|---|---|---|---|---|---|
| `presets[hosted].qualityFloor` | `placementPresets.mjs` | the recorded floor row | a run over other documents is not comparable (`documentsDigest`) | the constant's comment | `placementPresets.spec` |
| `presetStatus(hosted)` | same | `supported` | — | module JSDoc (unchanged) | spec arm |
| `referenceFloor()` / `REFERENCE_PRESET_ID` | `presetQualityFloor.mjs` | the bar is the named reference preset's floor (`local-small`), whichever presets carry floors | a table without the reference preset → `null` (nothing supported by default, unchanged) | the constant's JSDoc | instrument spec arm: hosted's row is not the reference and meets it |
| recipe recommendation | `firstRunRecipe.mjs#recommendPlacement` (unchanged) | hosted recommended where it fits with headroom | a host under the headroom keeps it possible-not-recommended | — | `firstRunRecipe.spec` placement arms |

Decision Record impact: none (the floor rule and its instrument are #714 / #746; this records a measurement).

## Acceptance Criteria

- AC-1: `presetStatus(presets.hosted)` reads `supported`; the row names the three receipts' documents, digest and the weakest met result.
- AC-2: `recommendPlacement` recommends `hosted` wherever it clears the 4 GiB headroom — the generous host (beside `local-small`), the bare host (where `local-small` stays possible by its own 1 GiB), the swapping host (the probe refuses only local presets), and the probe's 32 GiB fixture (hosted clears it by 13 GiB: the placement step reads "recommended: hosted"); a host with 1 GiB above the plane's own peak keeps hosted possible-not-recommended BY HEADROOM (the control: "fits by 1.0 GiB … under the 4.0 GiB headroom"); a floorless preset is still never recommended; the candidate wording no longer appears for hosted.
- AC-3: the three receipts are linked from the row's note (#746's ledger comments).

## Out of Scope

Re-measuring; the local presets' rows; the floor rule; the wizard's UI copy (it renders the recipe's reason). One instrument change IS in scope as the row's consequence: `referenceFloor()` names its reference preset (`REFERENCE_PRESET_ID = 'local-small'`) instead of taking the first preset carrying a floor — with hosted first in the table, the promoted candidate's row would otherwise become the bar for every later candidate; a recorded hosted floor is a receipt of meeting the bar, never the bar.

## Related

#746 (the ledger) · #777 (the rule) · #714 (the instrument) · #686 (the presets) · neomjs/neo-agent-institution#351 (parent)

Live latest-open sweep: latest 12 open Brain issues at 2026-10-03T07:44Z (#797 … #761), no equivalent; search "hosted floor / HOSTED_FLOOR / hosted preset supported": only the closed lineage above. A2A: Grace's 06:34Z rule names this leaf as mine on two met samples; no competing claim. Own-assignment: #50 #51 #53 #782 #784, none overlap. Structure map: existing module, no new file.

Origin Session ID: fb9561d9-a0dd-4f35-912c-095864afbae4
Retrieval Hint: "HOSTED_FLOOR hosted preset supported quality floor three consecutive samples gemini-3.8-flash"


## Timeline

- 2026-10-03T07:45:01Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-03T07:45:02Z @neo-fable-clio added the `enhancement` label
- 2026-10-03T07:45:02Z @neo-fable-clio added the `ai` label
- 2026-10-03T07:45:03Z @neo-fable-clio added the `agent-os` label
- 2026-10-03T07:45:10Z @neo-fable-clio added parent issue #351
- 2026-10-03T07:49:44Z @neo-fable-clio cross-referenced by PR #799
- 2026-10-03T08:21:07Z @neo-fable-clio cross-referenced by #746
- 2026-10-03T08:21:24Z @neo-fable-clio referenced in commit `8fcbbe6` - "fix(fleet): the hosted floor's note keeps the failed first sample and links each receipt; a below-headroom hosted control (#798)

Euclid's review precheck: sample 1 ran under the identity rule and failed on one genuine invention —
the note said otherwise; the three receipts are now linked from the row; the recipe spec gains the
host where hosted fits by 1 GiB and stays possible, not recommended — by headroom, never by status."
- 2026-10-03T08:45:00Z @tobiu referenced in commit `321bc1e` - "feat(fleet): the hosted preset records its quality floor and becomes supported (#798) (#799)

* feat(fleet): the hosted preset records its quality floor and becomes supported (#798)

Three consecutive met samples under the identity rule (5-7 / 4-8 / 4-5 grounded nodes per document,
0 dangling, 0 ungrounded; receipts on the readiness-probe ticket) put HOSTED_FLOOR beside GEMMA_FLOOR,
recording the WEAKEST met result. presetStatus(hosted) reads supported, so a host that clears the
headroom is recommended the hosted preset instead of being told it is a candidate.

The instrument's reference floor is now the named reference preset (local-small, the gemma run), not
the first preset carrying a floor: a promoted candidate's row is a receipt of meeting the bar, never
the bar.

* fix(fleet): the hosted floor's note keeps the failed first sample and links each receipt; a below-headroom hosted control (#798)

Euclid's review precheck: sample 1 ran under the identity rule and failed on one genuine invention —
the note said otherwise; the three receipts are now linked from the row; the recipe spec gains the
host where hosted fits by 1 GiB and stays possible, not recommended — by headroom, never by status."
- 2026-10-03T08:45:00Z @tobiu closed this issue

