---
id: 480
title: Provider responses are accepted without checking the served model
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-preview
createdAt: '2026-09-25T10:07:29Z'
updatedAt: '2026-09-25T22:05:15Z'
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
closedAt: '2026-09-25T22:05:15Z'
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
- 2026-09-25T15:33:45Z @neo-preview referenced in commit `4c4994c` - "fix(provider): keep served-model identity action-free (#480)"
- 2026-09-25T15:36:00Z @neo-preview referenced in commit `b231c34` - "merge(dev): sync #480 branch with current dev (#480)

# Conflicts:
#	.github/workflows/brain-unit.yml"
- 2026-09-25T16:44:49Z @neo-preview cross-referenced by PR #501
- 2026-09-25T17:01:29Z @neo-preview cross-referenced by PR #502
- 2026-09-25T17:41:16Z @neo-preview cross-referenced by #503
- 2026-09-25T18:50:44Z @tobiu referenced in commit `c1c9568` - "fix(provider): require the declared provider's HTTPS origin, not its hostname alone (#480)

Round 2, RA-1. @neo-gpt caught this by reading rather than by a test, and
they were right: replacing the host-shape heuristic with a declared-contract
lookup also dropped the protocol check that heuristic had been carrying.

`hostedModelAliasesAllowed` tested only hostname membership, so
`http://api.openai.com/v1` returned true and inherited a date-alias tolerance
it never earned. A declared provider's origin is `https://` AND the hostname;
the same hostname over plaintext is a different endpoint — unauthenticated as
that provider and unprotected in transit — so a date-suffixed served id
arriving from one is precisely the wrong-resident signal this whole assertion
exists to catch. A gate that gets stricter about trust while silently
loosening the transport is worse than the shape heuristic it replaced.

Falsified before believing it, and the first probe was no good: removing the
condition with a text substitution that spanned lines corrupted the function
and failed three assertions with `undefined`, which is a broken probe rather
than a result. Redone as a single-line condition swap: exactly one arm flips,
`Expected: false, Received: true` on the plaintext host, 17 of 18 still green.
The distinction matters — the first result would have been reported as
evidence and would have meant nothing.

116/116 focused at this head."
- 2026-09-25T19:20:46Z @tobiu referenced in commit `2846347` - "fix(provider): make the collection witness valid, and prove it discriminates (#480)

Round 2, RA-2, first half. @neo-gpt was right that the witness was
non-discriminating, and the reason is worth stating precisely: my response
omitted `index: 0`, so the dense-index guard refused the batch BEFORE the
identity check was reached. The zero-upsert assertion then held for the wrong
reason — it would have passed with the served-model guard bypassed entirely.

Two changes make the witness mean what it claims:

- The mismatch response is now a VALID embedding reply (`index: 0` present),
  so the only thing that can refuse it is the identity check.
- A new CONTROL arm serves an AGREEING model with the same well-formed shape
  and asserts the vector DOES reach the collection. This is the non-vacuity
  control the arm was missing: without it, a response refused by any earlier
  guard produces the same zero, and the assertion cannot tell the difference.

Falsified both directions rather than asserted. Disabling the identity
assertion at the embedding call site flips EXACTLY ONE arm — the mismatch arm,
`expected MODEL_MISMATCH as the batch-abort cause, got undefined` — while the
control stays green. So the mismatch arm now fails for the right reason, which
is the whole of RA-2's first ask.

The ADR 0019 B4 half is NOT done and is deliberately not faked. The two
`aiConfig.openAiCompatible.host` writes stay for now, and the reason is a
finding rather than an oversight: the host is read inline at three sites in
TextEmbeddingService with no resolver seam, so B4-compliant isolation cannot be
reached by test-only changes — it needs a small production seam. `node
ai/scripts/lint/check-aiconfig-test-mutation.mjs` reports 0 new violations
across 825 scanned test files WITH those writes present, so the guard does not
see them either. That is a scope fork for the reviewer who wrote the action,
and it goes to @neo-gpt with this evidence rather than being resolved here.

117/117 focused at this head."
- 2026-09-25T19:53:19Z @tobiu referenced in commit `b182ac2` - "feat(memory-core): resolve the OpenAI-compatible host through an injectable seam (#480)

ADR 0019 B4 half of RA-2. The identity-guard arm needed a real HTTP endpoint
to talk to, and the only way to point this service at one was to write
`aiConfig.openAiCompatible.host` on the shared singleton and restore it after.
B4 calls that safety-critical, and the reason is structural rather than
stylistic: the write lands on shared state, so a missed cleanup, a shared
process or plain test order hands the next consumer the test's endpoint. An
isolation mechanism that is itself a cross-test hazard is not isolation.

The seam follows `MaintenanceBackpressureService.resolveConfiguredTenantRepoLabelsFn_`
so the pattern is the codebase's, not this file's invention — and that
precedent is also the correction: Neo's `setupClass()` strips the trailing
underscore from a `static config` key, so the instance member is
`openAiCompatibleHostFn`. Reading it with the underscore left it `undefined`,
which surfaced as `this.openAiCompatibleHostFn is not a function` in a spec
that never assigned it.

Two things hid here, and both are the same mistake: I grepped for
`aiConfig.openAiCompatible.host` and treated three hits as the surface. The
site that actually issues the request reads the host by DESTRUCTURING —
`const {host} = aiConfig.openAiCompatible` — which binds a bare `host` and
never reads as a member expression. The arm failed with a `MODEL_MISMATCH`
cause of `undefined` because the request was still going to the configured
host, not the fixture. The seam is only real once the transport uses it.

Scope note, stated rather than implied: only the host leaf is seamed.
`TextEmbeddingService.retry.spec` mutates the same leaf plus `unloadRetryCount`,
`unloadRetryDelayMs` and `embeddingModel`, and a per-leaf seam is the wrong
general answer there — a real B4 fix for that spec wants an injected config
snapshot, which is a different change in a different ticket. Reported, not
silently widened.

The guard reports 825 files / 0 new violations, but it never flagged the
writes it exists to catch, so it is not evidence either way here; the absence
of shared-config writes in the spec is.

22/22 on the arm's spec; 76/76 with the retry spec. The three Ollama cap
failures and the `DatabaseService shared core sync` failure in a wide run are
pre-existing — the latter is gated on a clean checkout, so a one-blank-line
change in an unrelated file reproduces it and it clears on commit."
- 2026-09-25T20:19:45Z @tobiu cross-referenced by #508
- 2026-09-25T21:04:25Z @neo-opus-vega cross-referenced by PR #510
- 2026-09-25T22:05:15Z @tobiu referenced in commit `5284dca` - "Merge pull request #501 from neomjs/agent/480-served-model-identity-successor

feat(provider): enforce served-model identity (#480)"
- 2026-09-25T22:05:15Z @tobiu closed this issue

