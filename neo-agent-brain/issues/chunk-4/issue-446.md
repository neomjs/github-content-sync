---
id: 446
title: Tri-Vector schema union types break LM Studio MLX structured output for non-Gemma models; reasoning-only streams read as a silent empty response
state: CLOSED
labels: []
assignees:
  - neo-fable-clio
createdAt: '2026-09-23T19:17:50Z'
updatedAt: '2026-09-24T13:12:39Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/446'
author: neo-fable-clio
commentsCount: 1
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
closedAt: '2026-09-24T13:12:39Z'
---
# Tri-Vector schema union types break LM Studio MLX structured output for non-Gemma models; reasoning-only streams read as a silent empty response

## Summary

`SemanticGraphExtractor`'s Tri-Vector JSON schema declares two nullable fields as a union type — `feature_namespace: {type: ['string', 'null']}` and `roadmap_impact: {type: ['string', 'null']}` (`ai/services/graph/SemanticGraphExtractor.mjs`, the `triVectorSchema` literal inside `executeTriVectorExtraction`). LM Studio's MLX structured-output engine rejects that shape for every model tried except Gemma 4: the SSE stream carries `event: error` → `Error in iterating prediction stream: ValueError: 'type' must be a string`, and the provider hands the extractor an empty body, which the guardrail files as `under-band-choke / context-overflow` with the note "Silent empty-response from provider (no thrown error, no body)". Every REM extraction on such a model defers, forever, with a misleading reason.

## Evidence (2026-09-23, this host, LM Studio, brain dev)

Measured through the Brain's own path (`SemanticGraphExtractor.executeTriVectorExtraction`, provider from `NEO_*` env, payload captured before commit) on three real session documents (4.3k–10.4k chars):

| model (LM Studio key) | as shipped | with `type: [T, 'null']` rewritten to `anyOf` |
|---|---|---|
| `google/gemma-4-26b-a4b` (MLX 4-bit) | 3/3 payloads, 4–5 nodes, 0 dangling edges | unchanged |
| `gpt-oss-20b` (MXFP4-Q8) | 0/3 — LM Studio `ValueError: 'type' must be a string`, returned within 200 ms (no generation) | 3/3 payloads (1–3 nodes) |
| `qwen3.6-35b-a3b` (MLX 4-bit) | 0/3 — same error | 0/3 — see the second finding |

Raw stream captured through a local proxy (request unchanged, response): `data: {"error":{"message":"Error in iterating prediction stream: ValueError: 'type' must be a string"}}`. The same request with the `anyOf` form returns a normal completion.

## Second finding, same surface

For `qwen3.6-35b-a3b` LM Studio streams the model's ENTIRE output — including the JSON answer — as `delta.reasoning_content`; `delta.content` stays empty until `finish_reason: length`. None of the four request-level switches changes it (`chat_template_kwargs.enable_thinking=false`, `reasoning_effort: 'none'`, `reasoning: {enabled: false}`, a `/no_think` system line — all four: ~1,700 reasoning chars, empty content). That part is LM Studio's Qwen3.6 template handling, not the Brain's — but `OpenAiCompatible#getChoiceContent` reads `delta.content` only, so a reasoning-only stream becomes the same "silent empty-response" as a schema error. The provider should surface it as a typed failure (`reasoning-only-response`, with the reasoning byte count) instead of an empty body, so the guardrail can name the cause.

## Acceptance criteria

- AC-1: the Tri-Vector schema expresses nullable fields with `anyOf: [{type: 'string'}, {type: 'null'}]` (or an equivalent single-`type` form); a unit arm asserts no `type` array anywhere in the emitted `response_format.json_schema.schema`.
- AC-2: `gpt-oss-20b` on LM Studio (MLX) produces a Tri-Vector payload through `executeTriVectorExtraction` without a harness-side rewrite (the run above, re-executed on the fix).
- AC-3: a reasoning-only SSE stream (only `delta.reasoning_content` / `delta.reasoning` frames, empty `content`) is reported by the provider as a typed failure carrying the reasoning length, and the extractor's friction symptom names it rather than `context-overflow`.
- AC-4: Gemma 4's payloads are unchanged (the existing schema-vocabulary arms stay green).
- AC-5 (added 2026-09-24, same seam): an error frame the provider sends inside a 200 stream (`data: {"error": {"message": …}}` — the schema refusal above, as captured) is reported as a typed failure carrying the provider's message, never as an empty body.

## Contract Ledger

| Surface | Authority | Delivered behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Tri-Vector schema `feature_namespace` / `roadmap_impact` | `SemanticGraphExtractor.executeTriVectorExtraction` (`triVectorSchema`) | `{anyOf: [{type: 'string'}, {type: 'null'}]}`; null and string both valid, other types rejected | none — the schema is the provider request | schema comment | extractor spec "spells its nullable fields as anyOf" (walks the captured `responseSchema`, Ajv) |
| `OpenAiCompatible.stream()` / `generate()` ending with reasoning bytes and no content | producer `createReasoningOnlyResponseError` (`ai/provider/createStreamFailureError.mjs`), thrown by `stream()` after the frame gate | throws `Error` with `code: 'REASONING_ONLY_RESPONSE'`, `provider`, `reasoningBytes` (exact UTF-8 bytes of the reasoning text, counted once per logical frame), `finishReason`; the message carries the byte count, never the reasoning text | none — consumers that ignore `code` see a thrown `Error` where they saw `''` | module JSDoc | provider spec: reasoning-only SSE, `generate()`, byte count once for compact / compact+newline / pretty JSON / SSE |
| `OpenAiCompatible.stream()` receiving an error frame inside a 200 response | producer `createProviderStreamError`, thrown from the frame gate | throws `Error` with `code: 'PROVIDER_STREAM_ERROR'`, `provider`, `providerMessage` (≤ 300 provider-authored characters) | none | module JSDoc | provider spec: the captured LM Studio refusal as the fixture |
| `onProviderChunk` frame (`generate` / `stream` option) | `OpenAiCompatible#describeFrame` | `{content, reasoning, finishReason, error, raw}` — `reasoning` (`''` when absent) and `error` (`null` when absent) are additive; each logical frame is delivered once, the whole-body fallback runs only when no line parsed | callers reading only `content` / `finishReason` are unaffected | `#describeFrame` JSDoc | provider spec: reasoning-then-content control (one reasoning frame), byte-count arms (one callback per frame) |
| Empty stream (no content, no reasoning) | `stream()` | unchanged: yields nothing, `generate()` resolves `{content: ''}`; the extractor's silent-empty branch still owns that ending | — | — | provider spec control arm |
| Friction symptom vocabulary | `consumerFrictionHelper` `VALID_SYMPTOMS` / `DETERMINISTIC_SYMPTOMS` | `reasoning-only-response` added, deterministic (surfaces on the first emission), `suggestionKind` `unknown`; `categorizeInvocationError` maps `REASONING_ONLY_RESPONSE` structurally before the message regex | unknown codes fall through to the regex as before | typedef + JSDoc | helper spec: categorizer, deriveSuggestionKind, deterministic surfacing |
| Extractor defer reason | `SemanticGraphExtractor.getDeferReasonForFrictionSymptom` | `reasoning-only-response` → deferReason `reasoning-only-response`, `terminalForCadence: true`; `RemDigestion` persists it as a free string | other symptoms unchanged (`under-band-choke` default) | method JSDoc | extractor spec "filed as reasoning-only-response, not context-overflow" (descriptor, friction, one invocation) |
| `invokeWithGuardrail` note policy | `consumerFrictionHelper.invokeWithGuardrail` catch path, predicate `isProviderStreamFailureCode` | the error tail joins the caller note ONLY for `REASONING_ONLY_RESPONSE` / `PROVIDER_STREAM_ERROR`; untyped errors, `WITH_TIMEOUT`, `PROVIDER_TIMEOUT` and system codes keep the caller note, and the tail stands in only when no note was given (pre-existing policy) | — | inline comment + predicate JSDoc | helper spec: note-override control, reasoning-only join, foreign-code / timeout / stream-error controls |

## Context

Surfaced by the D#18965 (first-run journey) chat-model measurement for the local presets: https://github.com/neomjs/neo/discussions/18965 — the measured row will cite this ticket. Harness and captures live outside the repo (session scratchpad); the capture proxy is 30 lines and can be added to `ai/scripts/diagnostics` if a reviewer wants it.

Authored by Clio (Claude Fable 5.1, Claude Code). Session f34cbeb6-fd44-4060-b31f-e05332e62aee.




## Timeline

- 2026-09-23T19:17:52Z @neo-fable-clio assigned to @neo-fable-clio
### @neo-fable-clio - 2026-09-23T19:20:56Z

**Sunset handover (2026-09-23 evening, owner @neo-fable-clio, session f34cbeb6-fd44-4060-b31f-e05332e62aee) — not started in code; evidence complete.**

Pickup protocol for the next session (mine unless a REM-pipeline owner claims it first):
1. `ai/services/graph/SemanticGraphExtractor.mjs`, the `triVectorSchema` literal inside `executeTriVectorExtraction`: replace `feature_namespace: {type: ['string', 'null']}` and `roadmap_impact: {type: ['string', 'null']}` with `{anyOf: [{type: 'string'}, {type: 'null'}]}`. Keep the prose in the system instruction ("String … or null") as is — only the schema changes.
2. Unit arm beside the schema-vocabulary arm in `test/playwright/unit/ai/services/graph/SemanticGraphExtractor.spec.mjs` (it already captures `options.responseSchema` through a stubbed `OpenAiCompatible.prototype.generate`): walk the captured schema and assert no `type` value is an array; keep the Ajv validation + the three invalid-enum rejections green.
3. Provider (AC-3): `ai/provider/OpenAiCompatible.mjs` `#getChoiceContent` — count `delta.reasoning_content` / `delta.reasoning` bytes in `stream()`; when the stream ends with empty content but non-zero reasoning bytes, throw a typed error (`reasoning-only-response`, with the byte count) so the extractor's guardrail names it instead of `context-overflow`. A focused unit arm with an SSE fixture (frames carrying only `reasoning_content`) covers it.
4. Re-run the harness for AC-2: `/Users/Shared/clio/probes/d18965-2026-09-23/models/ab-run.sh gpt-oss-20b <label>` with the model loaded in LM Studio (`lms load gpt-oss-20b --context-length 65536 --identifier gpt-oss-20b -y`) — WITHOUT `AB_SCHEMA_FIX`; 3/3 payloads expected. The capture proxy (`capture-proxy.mjs`, 1235 → 1234) shows LM Studio's raw SSE if anything still fails.

Empirical anchors, all in `/Users/Shared/clio/probes/d18965-2026-09-23/models/`: `ab-gemma4-26b-a4b.json` (3/3), `ab-gpt-oss-20b.json` (0/3 as shipped), `ab-gpt-oss-20b-fix.json` (3/3 with the rewrite), `ab-qwen3.6-fix.json` (0/3, reasoning-only). No latency numbers are trustworthy from today — the host was swapping (operator's call).

Brain gates to remember before pushing: full unit tree of the touched services, `--update-parity` only if a config leaf changes (none here), the archaeology checker on the committed tree, `check-pr-body` on the PR body, cross-family reviewer (GPT seat).

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session f34cbeb6-fd44-4060-b31f-e05332e62aee


- 2026-09-24T11:08:20Z @neo-fable-clio cross-referenced by PR #447
- 2026-09-24T12:55:13Z @neo-fable-clio referenced in commit `6f0d25f` - "feat(provider): a stream that ends without an answer fails typed, and the Tri-Vector schema drops its type unions (#446)

The Tri-Vector schema spelled its two nullable fields as a union type, which LM Studio's MLX structured-output engine refuses for every model but Gemma 4 with an error frame inside a 200 stream; the OpenAI-compatible transport read only the content channel, so that frame, and a Qwen3.6 answer streamed entirely as reasoning_content, reached the extractor as an empty string it filed as context-overflow. The schema now uses anyOf for both fields. The transport routes every frame through one gate: an in-stream error throws PROVIDER_STREAM_ERROR carrying the provider's message, and a stream that ends with reasoning bytes but no content throws REASONING_ONLY_RESPONSE carrying the byte count, both from a sibling of the timeout contract. The friction helper classifies the latter as the deterministic symptom reasoning-only-response and keeps a typed failure's message beside the caller's note; the extractor maps it to its own defer reason instead of under-band-choke. The Brain unit smoke list executes the provider and friction-helper specs."
- 2026-09-24T12:55:14Z @neo-fable-clio referenced in commit `f0523ad` - "fix(provider): a compact JSON body counts its reasoning once, and only provider stream endings join the friction note (#446)

A compact JSON body parsed as the stream's final line and then again through the whole-body fallback, so its reasoning bytes doubled and its frame callback fired twice; the fallback now runs only when no line parsed, which is the case it existed for. The guardrail's note policy read any truthy error code as a provider contract; it now asks the stream-failure module's own predicate, so a foreign coded error such as WITH_TIMEOUT or a timeout keeps the caller's note as before. Both changes carry their controls: the byte count for compact, compact-with-newline, pretty JSON and SSE bodies, and the note for foreign, timeout and stream-error codes."
- 2026-09-24T13:12:39Z @tobiu referenced in commit `e2135b2` - "Merge pull request #447 from neomjs/clio/446-schema-anyof-reasoning-only

feat(provider): a stream that ends without an answer fails typed, and the Tri-Vector schema drops its type unions (#446)"
- 2026-09-24T13:12:39Z @tobiu closed this issue

