---
id: 744
title: The hosted preset routes graph generation through Gemini's OpenAI-compatible endpoint
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-10-02T09:26:33Z'
updatedAt: '2026-10-02T16:57:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/744'
author: neo-fable-clio
commentsCount: 1
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
closedAt: '2026-10-02T11:53:28Z'
---
# The hosted preset routes graph generation through Gemini's OpenAI-compatible endpoint

## Context

Found by the quality-floor instrument (#714 → PR #743) on 2026-10-02: the hosted preset in `ai/services/fleet/placementPresets.mjs` declares `NEO_GRAPH_PROVIDER: 'gemini'`, and graph generation refuses it. `ai/services/graph/providerDispatch.mjs` dispatches graph work to `ollama` or an OpenAI-compatible endpoint only (`GRAPH_MODEL_PROVIDERS = ['ollama', 'openAiCompatible']`, "deliberately"), and `buildGraphProvider` fails loud on anything else. The instrument therefore reports the hosted preset `unmeasured: graph provider 'gemini' is outside the Tri-Vector dispatch` before any request. Parent: neomjs/neo-agent-institution#351; blocks #714's AC-3 (the hosted floor run).

## The Problem

A plane provisioned with the hosted preset — the only preset the recipe can offer on the 32 GiB tier, the likeliest outside host — would summarize and embed through Gemini and extract NO graph: the dream pipeline's Tri-Vector synthesis, the Golden Path synthesizer and the topology inference all go through the same dispatch, which refuses the preset's graph provider. The preset table is the surface that must say which endpoint graph generation uses; today it says one the engine does not accept.

## The Architectural Reality

- `providerDispatch.resolveGraphModelProvider` returns the configured `graphProvider` verbatim and `buildGraphProvider` builds an `OpenAiCompatible` or `Ollama` client from `AiConfig.openAiCompatible` / `AiConfig.ollama`; the chat and embedding paths keep their own Gemini clients. This split is the dispatch's documented design, not a gap to patch there.
- Gemini serves an OpenAI-compatible endpoint (`https://generativelanguage.googleapis.com/v1beta/openai`) that takes the same bearer key; `OpenAiCompatible` already reads its key through the two carriers (#736), so the hosted key file the credential step writes can feed `openAiCompatible.apiKeyFile` beside `geminiApiKeyFile`.
- The local overlay feeds the `openAiCompatible` leaves from its `NEO_LOCAL_AGENT_OS_*` inputs and lets a preset choose the three providers (#715); a preset's env must be inputs the profile reads (the parity witness `unconsumedPresetEnvKeys`).
- The Tri-Vector call uses `json_schema` structured output and `reasoning_effort` through the OpenAI-compatible client. Gemini's documentation (read 2026-10-02) lists both for the compatible endpoint, with the `reasoning_effort` value `none` — the leaf's default, the local gemma's no-think pass — for 2.5 models only.
- Measured 2026-10-02 against Google: a wrong bearer answers 400 "Please pass a valid API key" on `/chat/completions` AND on the client's `/v1/chat/completions` (a nonsense path answers 404) — the client's `${host}/v1/…` suffix is routed, so no client change is needed; the keyless `GET /v1/models` answers 404, which is the readiness probe's problem (#746), not this ticket's.

## The Fix

1. The hosted preset's env routes graph generation through the OpenAI-compatible leaves over the overlay's own inputs — `NEO_GRAPH_PROVIDER: openAiCompatible`, `NEO_LOCAL_AGENT_OS_PROVIDER_HOST: https://generativelanguage.googleapis.com/v1beta/openai`, `NEO_LOCAL_AGENT_OS_MODEL: gemini-3.5-flash` — with chat and embeddings staying `gemini`.
2. The credential step (#696) writes the Gemini key once and emits three `_FILE` env values for the hosted preset: `NEO_GEMINI_API_KEY_FILE`, `GEMINI_API_KEY_FILE` and `NEO_OPENAI_COMPATIBLE_API_KEY_FILE` (the last two the same mount); the overlay forwards `NEO_OPENAI_COMPATIBLE_API_KEY_FILE` as an input, so the parity witness stays green.
3. The overlay forwards `NEO_LOCAL_MODELS_CHAT_GRAPH_REASONING_EFFORT` and the hosted preset sets `low` (Gemini's 3.x models take no `none`); a local plane keeps the leaf's default.
4. The instrument (#714) drops its own copy of the dispatch's list: the table's parity arm proves every preset's graph provider is one the dispatch serves, and the child's refusal stays the runtime answer.
5. A one-request premise check recorded on this ticket: `presetQualityFloor.mjs --preset hosted` with the operator's key in the environment (`NEO_OPENAI_COMPATIBLE_API_KEY_FILE`) returns a schema-valid payload at `reasoning_effort: low` (post-merge of #743's instrument).

## Contract Ledger

| Target surface | Authority | Behavior | Edge / refusal | Docs | Evidence |
|---|---|---|---|---|---|
| `presets.hosted.env` | #686's contract; `providerDispatch` | graph generation declares an OpenAI-compatible endpoint (Gemini's), chat + embeddings stay Gemini | a graph provider outside the dispatch never ships in the table (a parity arm) | module JSDoc + `ModelProviders.md` | AC-1 |
| the credential step's hosted outputs | #696 | one key file, three `_FILE` env values | a missing key refuses before any write (existing) | JSDoc | AC-2 |

## Acceptance Criteria

- [ ] AC-1 The hosted preset's `NEO_GRAPH_PROVIDER` is a value `GRAPH_MODEL_PROVIDERS` accepts, its graph endpoint, model and reasoning effort are inputs the profile reads, and a parity arm fails any preset whose graph provider is outside the dispatch. Unit.
- [ ] AC-2 The credential step emits the hosted preset's three key-file env values from one written file; the carrier holds paths only. Unit (the existing credential-step spec extended).
- [ ] AC-3 *(post-merge, operator key)* `presetQualityFloor.mjs --preset hosted` reaches a measured result (not `unmeasured`); the receipt is recorded as #746's AC-4 (the last open leaf of the hosted lane — this ticket closes with its PR, #714 with #743) and mirrored on neomjs/neo-agent-institution#351.

## Out of Scope

Teaching `providerDispatch` a native Gemini graph client (the split is its design); the readiness probe's missing bearer (#746); the floor's value for hosted (#714 AC-3 records it); the local presets.

## Avoided Traps

Patching the dispatch to accept `gemini` silently (three graph consumers share it; the doc says deliberate). A second key file for the same credential. Calling the hosted preset `supported` before the hosted run. Reading the keyless 404 on `/v1/models` as "unrouted" — the bearer probe shows the path is served.

## Related

#714 / PR #743 (the instrument that found it) · #686 / PR #715 (the table) · #696 / PR #736 (the carriers) · #746 (the readiness probe's bearer, found on the way) · `ai/services/graph/providerDispatch.mjs` · neomjs/neo-agent-institution#351 (parent)

Decision Record impact: aligned-with ADR 0019 (a preset is env over declared leaves; the dispatch reads AiConfig at the use site).

Sweeps: live latest-open sweep — the latest 20 open Brain issues at 2026-10-02T09:25Z (newest #741), no equivalent; search over all states for the hosted preset's graph provider returned nothing; A2A in-flight sweep — the last 30 messages at 09:24Z, no claim on this surface; Memory Core rationale sweep — the 2026-09-23 chat-model measurements (`d18965-chat-model-measured`) ran gemma through the shipped path and named "a hosted row (Gemini Flash)" as the 32 GB tier's answer without reaching the graph dispatch; own-assignment sweep — #714, #37, #50, #51, #53, none overlapping; structure map — the presets table, the overlay, the credential step and the instrument, no new file.

Edit 2026-10-02 10:10Z (author, before the PR): the probe facts folded in (routing measured with a wrong bearer; `reasoning_effort: none` is 2.5-only per Gemini's documentation → Fix 3; the readiness probe split out as #746); AC-1 names the reasoning-effort input, AC-2 counts three env values.

Origin Session ID: 1efa16ff-bd83-41e5-87dc-4c186b03b451
Retrieval Hint: "hosted preset graph provider gemini openAiCompatible endpoint Tri-Vector dispatch refused unmeasured"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 1efa16ff-bd83-41e5-87dc-4c186b03b451


## Timeline

- 2026-10-02T09:26:34Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-02T09:26:35Z @neo-fable-clio added the `enhancement` label
- 2026-10-02T09:26:35Z @neo-fable-clio added the `ai` label
- 2026-10-02T09:27:15Z @neo-fable-clio added parent issue #351
- 2026-10-02T09:36:15Z @neo-gpt-emmy cross-referenced by #430
- 2026-10-02T10:05:27Z @neo-fable-clio cross-referenced by #746
- 2026-10-02T10:10:21Z @neo-fable-clio cross-referenced by PR #747
- 2026-10-02T10:37:21Z @neo-gpt cross-referenced by PR #743
- 2026-10-02T10:57:23Z @neo-fable-clio referenced in commit `e828ea6` - "feat(fleet): the hosted preset routes graph generation through Gemini's OpenAI-compatible endpoint, its key and reasoning effort over the overlay's inputs (#744)"
- 2026-10-02T11:16:29Z @neo-fable-clio referenced in commit `17726fa` - "feat(fleet): the hosted preset routes graph generation through Gemini's OpenAI-compatible endpoint, its key and reasoning effort over the overlay's inputs (#744)"
- 2026-10-02T11:36:50Z @neo-fable-clio referenced in commit `f61f378` - "feat(fleet): the hosted preset routes graph generation through Gemini's OpenAI-compatible endpoint, its key and reasoning effort over the overlay's inputs (#744)"
- 2026-10-02T11:53:28Z @tobiu referenced in commit `40ed1dd` - "feat(fleet): the hosted preset routes graph generation through Gemini's OpenAI-compatible endpoint, its key and reasoning effort over the overlay's inputs (#744) (#747)"
- 2026-10-02T11:53:28Z @tobiu closed this issue
- 2026-10-02T12:21:30Z @neo-gpt-sophie cross-referenced by PR #748
- 2026-10-02T16:49:20Z @neo-fable-clio cross-referenced by #767
### @neo-fable-clio - 2026-10-02T16:57:23Z

**Hosted preset — the first floor runs through the instrument (2026-10-02, the operator's Gemini key as a file, never printed).**

Before the fix (Brain #767 / its PR): the first run failed in one second — Gemini's OpenAI-compatible endpoint refuses a request carrying `keep_alive` and `operationStage` (`400 Invalid JSON payload received. Unknown name …`); the client leaked both; LM Studio answers 200 to the same fields, which is why the local lane never saw it.

After the fix (`presetQualityFloor.mjs --preset hosted` on the #767 tree; `gemini-3.5-flash` over `https://generativelanguage.googleapis.com/v1beta/openai`, `reasoning_effort: low`, `json_schema` structured output; the child isolated: graph store `:memory:`, scratch anchor + marker dir; `documentsDigest f3cd8b71…16844e` = the table's):

| run | time | schemaValid | dangling | grounded / doc | ungrounded | comparable | met |
|---|---|---|---|---|---|---|---|
| 1 | 19 s | true | 0 | 3–5 | 2 (`Neo.main.DomEvents`, `Neo.dashboard.dock.Workspace`) | true | false |
| 2 | 23 s | true | 0 | 4–5 | 2 (`Focus Management`, `Neo.dashboard.dock.Workspace`) | true | false |

Reading: the hosted lane extracts a schema-valid graph with no dangling edge and more grounded claim nodes per document than the gemma reference (3–4); it reads below the reference on ungrounded names (gemma: 0) on both samples. The names are canonical class names the threads only imply (the thread says "dock Workspace" / `Workspace.mjs`; the model writes `Neo.dashboard.dock.Workspace`) and one concept label. By the floor rule (at or above on every recorded axis) `hosted` stays `candidate` and the 32 GiB tier keeps "nothing recommended"; the rule was reviewed as strict on purpose. Hypothesis, not acted on: word-wise grounding of a dotted name by its last segment would read both class names as grounded — a decision for the instrument's owner, with its own V-B-A (what else it would admit).

Calls spent: 10 on this lane today (one failing run, a model list, two shaped probes, two measured runs). The wire fix is Brain #767; the readiness probe's bearer is Grace's #748 (#746).

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 1efa16ff-bd83-41e5-87dc-4c186b03b451


- 2026-10-02T17:17:42Z @neo-fable-clio cross-referenced by #773
- 2026-10-02T17:26:19Z @neo-gpt cross-referenced by PR #770
- 2026-10-02T17:32:51Z @neo-fable-clio cross-referenced by #776
- 2026-10-02T17:55:41Z @neo-gpt cross-referenced by PR #775

