---
id: 773
title: Gemini Flash defaults move to gemini-3.8-flash (preset + provider)
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-10-02T17:17:40Z'
updatedAt: '2026-10-02T17:57:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/773'
author: neo-fable-clio
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
closedAt: '2026-10-02T17:57:03Z'
---
# Gemini Flash defaults move to gemini-3.8-flash (preset + provider)

Found on 2026-10-02 (the operator's note "gemini flash is already at version 3.8"): Neo pins `gemini-3.5-flash` in six places — the hosted placement preset (`ai/services/fleet/placementPresets.mjs:134`, `GEMINI_FLASH`, projected as `chatModel`, `NEO_GEMINI_MODEL`, `NEO_LOCAL_AGENT_OS_MODEL` and, since #747, the graph lane's `NEO_OPENAI_COMPATIBLE_MODEL`), the Gemini SSOT leaf default (`ai/configBase.mjs:1069`), the provider class default (`ai/provider/Gemini.mjs:21`), the Knowledge Base generation leaf (`ai/mcp/server/knowledge-base/configBase.mjs:694`), the demo agent (`ai/demo-agents/dev.mjs:37`) and `learn/agentos/ModelProviders.md:150`. Sibling of #744 / #746 / #767 under neomjs/neo-agent-institution#351.

## Context

V-B-A with the operator's key (never printed), 2026-10-02: `GET …/v1beta/openai/models` serves `gemini-3.8-flash`, `gemini-3.7-flash`, `gemini-3.6-flash`, `gemini-3.5-flash`, `gemini-3.5-flash-lite` and the aliases `gemini-flash-latest` / `gemini-flash-lite-latest`; there is no `gemini-3.8-flash-lite` (the 3.8 lite line is TTS only). The pricing page (ai.google.dev/gemini-api/docs/pricing, read the same day), per 1M tokens: `gemini-3.8-flash` $0.75 in / $3.75 out through 2026-12-31, then $1.50 / $7.50; `gemini-3.5-flash` $1.50 / $9.00; `gemini-3.5-flash-lite` $0.30 / $2.50. No retirement date for 3.5-flash is listed: this is a current-line and price move, not a cliff.

Quality floor (`presetQualityFloor.mjs`, hosted preset, measured from the #770 tree with `reasoning_effort: low`, three requests per sample, documents digest `f3cd8b71…` = the table's):

| model | grounded / document | dangling | ungrounded names | wall | floor |
|---|---|---|---|---|---|
| gemini-3.5-flash (2 samples, recorded on #770) | 3–5 · 4–5 | 0 | 2 · 2 | 19 s · 23 s | comparable, met: false |
| gemini-3.8-flash (1 sample) | 4–5 | 0 | 3 | 19 s | comparable, met: false |
| gemini-3.5-flash-lite (1 sample) | 2–5 | 0 | 0 | 11 s | comparable, met: false (2 < gemma's 3 on the grid thread) |

The 3.8 ungrounded names are one dotted canonical name per thread (`Neo.main.DomEvents`, `Neo.dashboard.dock.Workspace`, `Neo.grid.Body.updateMountedAndVisibleRows`); each last segment occurs in its thread (3 / 10 / 3 times) — the class #770 recorded for 3.5. The grounded count rises with 3.8; the strict rule fails both generations the same way, so the bump moves the hosted preset's `candidate` status in neither direction.

## The Problem

Every hosted-preset user pays the previous generation's price (2× of 3.8 on input, 2.4× on output through 2026) for the previous generation's model, and six pins say the same name without one of them being the decision.

## The Architectural Reality

- The preset's `GEMINI_FLASH` constant is the hosted preset's declaration (the placement projection names its env explicitly); the SSOT leaf `gemini.modelName` (`NEO_GEMINI_MODEL`) is the deployment default the preset overrides by name; the class default in `Gemini.mjs` is what an unconfigured provider instance reads. ADR 0019: a leaf default is a declared value read at the use site — a default change touches no §3 row (no re-derivation, no export, no runtime write).
- The floor instrument records `chatModel` per run; a floating alias (`gemini-flash-latest`) would change underneath a receipt.
- Structure map: `ai/services/fleet` owns `placementPresets.mjs`; no new file.

## The Fix

One name everywhere it is a pin: `GEMINI_FLASH = 'gemini-3.8-flash'`, the SSOT leaf default, the provider class default and its `@member` line, the KB generation leaf, the demo agent constant, `ModelProviders.md`. Specs that pin behaviour follow (`placementPresets.spec`, `presetQualityFloor.spec` env + summary expectations, `config.template.spec`, `configBase.spec`); fixtures that only need some model string (`firstRun.spec`, `SessionService.buildChatModel.spec`, the `declaredEnvBindings` string) move too. `initServerConfigs.spec`'s parser sample keeps the old literal: the string is a parsing input, not a pin, and touching that file inherits four pre-existing archaeology refs this leaf does not own. The floor table (`GEMMA_FLOOR`) stays untouched: hosted remains `candidate`; the 3.8 receipt lands on the PR and on #746.

## Contract Ledger

Added 2026-10-02 at the reviewer's precheck of PR #775. Authority is the existing owners and ADR 0019; the evidence is AC-1 – AC-3 as recorded.

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| the hosted preset: `GEMINI_FLASH` → `chatModel`, `NEO_GEMINI_MODEL`, `NEO_LOCAL_AGENT_OS_MODEL` (`ai/services/fleet/placementPresets.mjs`); `mapPresetEnv` → `NEO_OPENAI_COMPATIBLE_MODEL` (`ai/scripts/diagnostics/presetQualityFloor.mjs`) | the preset table (the fleet service); the floor rule (#714 / PR #743) | the projected strings read `gemini-3.8-flash`; the hosted preset's status stays `candidate` (no floor row of its own) | a placement that names its own model through the projected env keeps it; the floor compares per run | `learn/agentos/ModelProviders.md` | AC-1; the 3.8 floor receipt (Context above; #746 AC-4) |
| the Gemini defaults: the `gemini.modelName` SSOT leaf (`ai/configBase.mjs`, env `NEO_GEMINI_MODEL`), `Gemini#modelName` (`ai/provider/Gemini.mjs`), the Knowledge Base generation leaf (`ai/mcp/server/knowledge-base/configBase.mjs`, no env binding), `MODEL_NAME` (`ai/demo-agents/dev.mjs`) | ADR 0019 — a leaf default is a declared value, read at the use site; the owning config files | the defaults read `gemini-3.8-flash`; no leaf, env binding, parser or precedence changes | `NEO_GEMINI_MODEL` still overrides the SSOT leaf; a configured provider instance still overrides the class default | `learn/agentos/ModelProviders.md` names the default | AC-2; `lint-config-template-ssot`, `check-aiconfig-antipatterns`, `check-aiconfig-test-mutation` green; `config.template.spec` / `configBase.spec` |
| the guide and spec pins: `ModelProviders.md`, `placementPresets.spec`, `presetQualityFloor.spec`, `config.template.spec`, `configBase.spec`, `firstRun.spec`, `SessionService.buildChatModel.spec` | the specs' own expectations | every pin and fixture reads `gemini-3.8-flash`, except `initServerConfigs.spec`'s parser sample — a parsing input, and the file carries four pre-existing archaeology refs this leaf does not own | none needed: a pin has no runtime | the guide line itself | AC-2's grep — the parser sample is the only remaining literal |

## Acceptance Criteria

- AC-1: the hosted preset reads `chatModel: 'gemini-3.8-flash'` and projects it as `NEO_GEMINI_MODEL`, `NEO_LOCAL_AGENT_OS_MODEL` and, through `mapPresetEnv`, `NEO_OPENAI_COMPATIBLE_MODEL`; `placementPresets.spec` and `presetQualityFloor.spec` pin it.
- AC-2: the Gemini SSOT leaf, the provider class default, the KB generation leaf and the demo agent read `gemini-3.8-flash`; `config.template.spec` / `configBase.spec` pin the leaf; `lint-config-template-ssot` and `check-aiconfig-antipatterns` green; `grep -rn gemini-3.5-flash` over the tree (node_modules excluded) finds only the `initServerConfigs.spec` parser sample.
- AC-3: the PR records the 3.8 floor receipt (the sample above, or a fresh one if the instrument changes under it) and states that the hosted preset's status is unchanged; the floor table carries no hosted row. Post-merge: none.

## Out of Scope

- The grounding rule (a dotted canonical name grounded by its last segment) — #770's hypothesis, a separate decision on the strict floor.
- `gemini-3.5-flash-lite` for the graph lane: 0 ungrounded at a fraction of the price, but one sample under-extracts the grid thread (3 nodes), and the preset's single `chatModel` also serves chat and the agent OS, which the instrument does not measure. A leaf of its own once the lane has two samples.
- `gemini-flash-latest`: rejected — a receipt needs a pinned name.

## Related

#744 / PR #747 (the hosted graph lane) · #746 (the receipts' home, AC-4) · #767 / PR #770 (the wire fix the measurement needed) · #714 / PR #743 (the instrument) · neomjs/neo-agent-institution#351 (parent)

Decision Record impact: aligned-with ADR 0019 (a leaf default is a declared value; §3 / §5 re-read 2026-10-02).

Live latest-open sweep: the latest 20 open Brain issues read at 2026-10-02 ~17:30Z (#772 … #521), no equivalent. A2A claim sweep: no `[lane-claim]` on the preset or the Gemini defaults in the last 60 min. Memory Core sweep ("hosted preset pins an older gemini flash chat model"): the 2026-07-09 memory recommended the 2.5 → 3.5 bump ahead of the 2026-10-16 retirement; nothing on 3.8. Own-assignment sweep: #767, #51, #53, #50 — none overlap.

Origin Session ID: 1efa16ff-bd83-41e5-87dc-4c186b03b451
Retrieval Hint: "gemini-3.8-flash hosted preset GEMINI_FLASH pin quality floor receipt price"


## Timeline

- 2026-10-02T17:17:41Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-02T17:17:42Z @neo-fable-clio added the `enhancement` label
- 2026-10-02T17:17:42Z @neo-fable-clio added the `ai` label
- 2026-10-02T17:25:35Z @neo-fable-clio referenced in commit `80b3df6` - "feat(ai): Gemini Flash defaults move to gemini-3.8-flash (#773)"
- 2026-10-02T17:25:38Z @neo-fable-clio cross-referenced by PR #775
- 2026-10-02T17:26:16Z @neo-fable-clio cross-referenced by #746
- 2026-10-02T17:32:51Z @neo-fable-clio cross-referenced by #776
- 2026-10-02T17:57:03Z @tobiu referenced in commit `8a078f0` - "feat(ai): Gemini Flash defaults move to gemini-3.8-flash (#773) (#775)"
- 2026-10-02T17:57:03Z @tobiu closed this issue

