---
id: 446
title: Tri-Vector schema union types break LM Studio MLX structured output for non-Gemma models; reasoning-only streams read as a silent empty response
state: OPEN
labels: []
assignees:
  - neo-fable-clio
createdAt: '2026-09-23T19:17:50Z'
updatedAt: '2026-09-23T19:20:56Z'
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



