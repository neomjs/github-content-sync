---
id: 480
title: Provider responses are accepted without checking the served model
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-preview
createdAt: '2026-09-25T10:07:29Z'
updatedAt: '2026-09-25T14:54:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/480'
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
---
# Provider responses are accepted without checking the served model

## Context

Filed on @neo-fable-clio's ask (A2A 2026-09-24T15:16:57Z: "you file it — it is the provider/residency lane's leaf, next to #460 → #462"), from a receipt taken on the local plane on 2026-09-24T14:38:46Z with LM Studio's JIT loading off (A2A MESSAGE:8538e8b4): a `/v1/embeddings` request for an unloaded model (`nomic`) was answered with the resident model's vectors (Qwen3, 4096 dimensions), and a `/v1/chat/completions` request for an unloaded model (`gpt-oss-20b`) was answered by the resident `gemma`. Both responses were well-formed and carried the served model's id in their `model` field; nothing in the Brain read it.

Sweeps (2026-09-25T10:04Z): live latest-20 open issues — no equivalent (#461, the per-role residency readout, is the sibling, not this); A2A last 30 messages — no claim on this scope; Memory Core rationale sweep — 579bee50 (2026-08-17: the existing identity assert runs only on the LM Studio lane), 45416767 (Clio's 2026-09-24 note: zero reads of a response's `model` field in `ai/provider`, `knowledge-base`, `memory-core`); own-assignment sweep — none of my nine open Brain assignments cover it. Structure map: `ai/provider` holds the typed-failure precedent; no new file.

## The Problem

A provider can serve a different model than the one requested and say so in the response, and the Brain stores or reasons over the result anyway:

- the embedding lane indexes foreign vectors (a 4096-dim Qwen3 vector under a nomic request), so the admission geometry check downstream is the only thing standing between a misnamed model and a poisoned collection;
- the chat lane returns another model's text as if it were the configured one (summaries, dream, `ask`), with no signal in the healthcheck or the friction classifier.

The Brain already has a model-identity check, but it is out-of-band and lane-gated: `TextEmbeddingService` (`#shouldAssertOpenAiCompatibleEmbeddingContext`, ~:1149–1330 at dev@2d37186) probes `lms ps` and throws `not resident under its configured identifier`, and it runs only when `orchestrator.lms.enabled` is true AND the openAiCompatible host's port equals `orchestrator.lms.port`. llama.cpp, vLLM, Ollama's compatibility surface and hosted APIs never run it (the 2026-08-17 klarso finding). The response's own `model` field is the provider-agnostic identity signal, present on every OpenAI-compatible response, and it is read nowhere.

## The Architectural Reality

- `ai/services/memory-core/TextEmbeddingService.mjs` ~:1526–1530 (dev@2d37186): the `/v1/embeddings` body is `JSON.parse`d and `resolveOnce(result)`; `result.model` is never compared to `embeddingModel`.
- `ai/provider/OpenAiCompatible.mjs` `#describeFrame` (:280): reads `choices[0]` (delta / message content, `reasoning_content`); the frame's `model` is not read.
- `ai/provider/createStreamFailureError.mjs`: the typed-failure family (`REASONING_ONLY_RESPONSE`, `PROVIDER_STREAM_ERROR`, from #447) with the exported predicate the friction helper classifies on; the file's own note says the predicate is the only thing that crosses, never a shared Set.
- #462's `replacement-required` result (readiness on the LM Studio lane) diagnoses "the resident model is not the configured one" and, until #493 lands, prints the fix for a person; this ticket adds the request-boundary observation of the same condition. #493 owns the action for both and retires the operator line.

## The Fix

1. Read `model` at both parse sites and compare it to the requested id after the provider adapter's alias normalization (hosted APIs legitimately answer `gpt-4o` with `gpt-4o-2024-08-06`; LM Studio and llama.cpp echo the id verbatim). A served id that is present and does not resolve to the requested one raises a typed failure; an absent `model` field yields no verdict and logs once per process.
2. `createStreamFailureError.mjs` gains `MODEL_MISMATCH` (`{provider, lane, requested, served}`) beside the two existing codes, and the exported predicate admits it, so the friction helper classifies it deterministically.
3. The embedding path rejects before admission (no vector is stored); the chat path rejects the response (no summary, dream or `ask` text is produced from it).
4. The LM Studio mismatch carries a bounded diagnosis naming the requested and served models, and no text addressed to a person (no `lms unload` / `lms load` imperative, no "operator action"). This ticket creates no live-request-to-supervisor bridge: the orchestrator's own identity probe (`DeploymentStateBridgeService.mjs:1297`) is the transport, and #493 turns its `mismatch` and the readiness `replacement-required` into the heal (route in #493).

Decision Record impact: aligned-with ADR 0025 (this is the detect signal; ADR 0025 §2.1: a probe is a signal, never the actuator); the act half is #493 under ADR 0026. No config leaf is added; the configured model ids are read where they already are.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `createStreamFailureError.mjs` predicate + `MODEL_MISMATCH` code | `ai/provider/createStreamFailureError.mjs` (#447) | predicate true for the new code; error carries `{provider, lane, requested, served}` | none (additive) | JSDoc on the factory | unit spec: red arm on a fixture answering `model: "other"` |
| embedding admission | `TextEmbeddingService.mjs` parse site | mismatch → typed failure before any vector is stored | absent `model` → no verdict, one log line | JSDoc at the parse site | unit spec: fixture server, stored-vector count unchanged |
| chat frames | `OpenAiCompatible.mjs` `#describeFrame` | mismatch → typed failure, no text delivered | absent `model` → no verdict | JSDoc | unit spec: SSE + non-SSE fixtures |

## Acceptance Criteria

- [ ] A fixture `/v1/embeddings` answering `model: "other"` for a request of `model: "configured"` raises `MODEL_MISMATCH` with `{requested: "configured", served: "other"}`, and no vector reaches the collection (red-first: the arm fails at dev@2d37186 by storing the vector).
- [ ] The same for `/v1/chat/completions`, both the SSE frame path and the non-SSE body path.
- [ ] A hosted-style alias (`gpt-4o` → `gpt-4o-2024-08-06`) passes; an absent `model` field passes with one log line.
- [ ] The friction helper's predicate classifies `MODEL_MISMATCH` (spec arm on the exported predicate).
- [ ] On the LM Studio lane the typed `MODEL_MISMATCH` error carries a bounded diagnosis naming both the requested and served models and no operator-addressed text (spec arm asserts the absence of `lms unload` / `lms load` / "operator action"); the readiness `replacement-required` line is #493's to retire, not this ticket's to reproduce.
- [ ] The new spec files are on `brain-unit.yml`'s run list (Brain Unit collects the whole suite and executes a named list; #201).

## Out of Scope

- The heal itself, unloading the wrong resident and loading the pinned role: #493, under ADR 0026's actuator envelope (record-with-diagnosis + autonomous action). neomjs/neo#17079's rule that a routine readiness pass never evicts a configured resident stands; #493 acts on a diagnosed wrong resident, not on a readiness pass.
- The per-role residency readout in the healthcheck (#461) and the readiness repair (#460, shipped in #462).
- Reopening or broadening closed #310.
- Changing unrelated MCP response-wrapper behavior.
- Connecting live request errors to ProcessSupervisor readiness state: not needed, the orchestrator's identity probe re-observes the served set every snapshot cycle, and #493 routes that observation to the actuator.

## Avoided Traps

- Strict string equality on the served id: breaks every hosted alias, so the comparison goes through the adapter's normalization and the absent-field case is a non-verdict, not a failure.
- Extending the lane-gated `lms ps` assert to other providers: it needs a CLI per provider and still cannot see what one request was answered with.
- A shared mutable Set of failure codes: the precedent file explains why only the predicate crosses.

## Related

- #460 → #462 (readiness repair, `replacement-required`), #461 (per-role residency), #447 (typed stream failures), #201 (the run list).
- #493 (the heal: the LM Studio lane replaces a wrong resident itself; the producer → transport → actuator route).
- neomjs/neo#17079 (routine readiness never evicts; narrowed by #493 for the diagnosed case).

## Contract Amendment (2026-09-25)

The review falsified the original AC-5 bridge assumption. `ProcessSupervisorService` reads `operatorDiagnostic` only from a task-owned `postSpawn` readiness result (`ProcessSupervisorService.mjs:492-507`), while a live `TextEmbeddingService` request error has no supervisor channel. The typed mismatch therefore owns a bounded requested/served diagnosis with no operator-addressed text; the readiness replacement line is #493's to retire. No live-request-to-supervisor state bridge is introduced here; the route from either observation to the heal is #493's.

Corrected 2026-09-25 (@tobiu's ruling: Agent OS self-diagnoses and self-heals; a repair is never pointed at the operator). A first correction at 14:30Z was overwritten by the 14:31Z amendment above; this revision merges both.

Prescription checked: `ai/provider/createStreamFailureError.mjs` owns the typed failure shape; `ai/provider/OpenAiCompatible.mjs` and `ai/services/memory-core/TextEmbeddingService.mjs` own the two response-boundary call sites.

Origin Session ID: ec6c7966-ab2b-43d0-89cd-5ec2262b8424
Assigned to: @neo-preview
Retrieval Hint: "LM Studio silent model substitution served model field MODEL_MISMATCH"



## Timeline

- 2026-09-25T10:07:31Z @neo-opus-vega added the `bug` label
- 2026-09-25T10:07:31Z @neo-opus-vega added the `ai` label
- 2026-09-25T10:07:32Z @neo-opus-vega added the `agent-os` label
- 2026-09-25T13:58:08Z @neo-preview cross-referenced by PR #488
- 2026-09-25T14:27:36Z @neo-preview assigned to @neo-preview
- 2026-09-25T14:28:47Z @neo-opus-vega cross-referenced by #493
- 2026-09-25T14:32:39Z @neo-preview referenced in commit `e28f588` - "fix(provider): name served-model operator diagnostic (#480)"

