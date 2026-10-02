---
id: 767
title: 'The OpenAI-compatible client leaks call-site options and an Ollama keep_alive onto the wire; strict endpoints (Gemini''s compat layer, OpenAI) refuse the request'
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-10-02T16:49:18Z'
updatedAt: '2026-10-02T17:28:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/767'
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
closedAt: '2026-10-02T17:28:55Z'
---
# The OpenAI-compatible client leaks call-site options and an Ollama keep_alive onto the wire; strict endpoints (Gemini's compat layer, OpenAI) refuse the request

## Context

Found on 2026-10-02 by the first hosted quality-floor run (#744 AC-3 / #746 AC-4, the operator's key): `presetQualityFloor.mjs --preset hosted` returned `unmeasured: schema-failure` after one second with an empty error message. One request through the extractor's own provider construction showed the cause — Gemini's OpenAI-compatible endpoint answers `400 Invalid JSON payload received. Unknown name "keep_alive": Cannot find field.` and `Unknown name "operationStage": Cannot find field.` The extractor reads that 400 as `parse-failure` and stops. The local lane never saw it: LM Studio answers 200 to the same unknown fields (measured on the maintainer host). Sibling of #744 / #746 under neomjs/neo-agent-institution#351 (link set after creation).

## The Problem

`OpenAiCompatible.preparePayload` merges every option it did not recognise into the wire payload (`Object.assign(payload, clonedOptions)`) and always writes `keep_alive` from the provider's `keepAlive` (leaf `openAiCompatible.keep_alive`, default `-1`). Call sites pass Neo's own bookkeeping as options — `operationLabel`, `operationStage`, `priority`, `timeoutMs`, `signal`, `onProviderChunk`, `maxCompletionTokens`, `responseSchema*` — and the client already strips most of them; `operationStage` and `priority` are not stripped, and `keep_alive` is an Ollama field with no OpenAI meaning. Lenient servers (LM Studio, llama.cpp, vLLM) ignore unknown fields; strict ones (Gemini's compat endpoint, OpenAI, Mistral) refuse the whole request. So the hosted preset's graph lane — #747's reason to exist — cannot complete one extraction, and any operator pointing `NEO_OPENAI_COMPATIBLE_HOST` at a strict endpoint by hand hits the same wall.

## The Architectural Reality

- `preparePayload` (`ai/provider/OpenAiCompatible.mjs:73-165`) owns the wire shape; its strips are an explicit list, so the two missing keys are a gap in that list, not a design change. The final merge is what lets any future call-site key leak; the honest shape is: known request fields are placed, call-site keys are deleted by name, the merge stays for genuine OpenAI extras (`temperature`, `top_p`, `stop`, `seed`, `tools` …).
- `keep_alive` on this lane came with the Brain import (`11552b0`, #13) beside `ollama.keep_alive`; it is written on every request. LM Studio's OpenAI route answers 200 to it and to any unknown field (measured), so it does nothing there; the native Ollama provider keeps its own `keep_alive`. Three builders pass the leaf through (`buildChatModel.mjs:145,188,209`, `providerDispatch.buildGraphProvider`, the extractor's `buildConfiguredGraphProvider`).
- The leaf parser (`Env.parseKeepAlive`) returns `undefined` for an empty value, so an operator can only ever reach the default — a default of `-1` therefore means "always on the wire". A default of `null` means "only when configured", and the hosted preset needs no new input.

## The Fix

1. `preparePayload` deletes `operationStage` and `priority` beside its existing strips (one named list, `CALL_SITE_OPTION_KEYS`, so the next bookkeeping key lands in one place).
2. `keep_alive` crosses the wire only when the resolved `keepAlive` is not `null`; the `openAiCompatible.keep_alive` leaf's default becomes `null` (an operator who wants Ollama's semantics on this lane sets `NEO_OPENAI_COMPATIBLE_KEEP_ALIVE`); `buildChatModel` / `buildGraphProvider` pass-throughs are unchanged (they forward whatever the leaf resolves).
3. Specs: a red-first arm that the payload carries none of the call-site keys and no `keep_alive` by default, an arm that a configured `keepAlive` still ships, the existing KeepAlive arms adjusted to the default; the hosted floor run as the live evidence (three requests).

## Contract Ledger

| Target surface | Authority | Behavior | Edge / refusal | Docs | Evidence |
|---|---|---|---|---|---|
| `OpenAiCompatible.preparePayload` | the provider; the OpenAI chat-completions request shape | the wire payload carries request fields only: call-site bookkeeping never crosses; `keep_alive` only when configured | a lenient server sees the same request minus noise; a strict server now accepts it | JSDoc | AC-1, AC-2 |
| `openAiCompatible.keep_alive` leaf | ADR 0019 (leaf + env) | default `null` = not sent; a configured value is sent verbatim | empty env → default (not sent) | leaf JSDoc + `ModelProviders.md` | AC-1 |

## Acceptance Criteria

- [ ] AC-1 With default config the OpenAI-compatible payload contains no `keep_alive`, `operationStage`, `priority`, `operationLabel`, `timeoutMs`, `signal`, `onProviderChunk`, `maxCompletionTokens` or `responseSchema*` key; with `keepAlive` configured, `keep_alive` is sent with that value. Unit, red-first.
- [ ] AC-2 The extractor's request against Gemini's compat endpoint succeeds end to end: `presetQualityFloor.mjs --preset hosted` returns a measured result (the hosted receipt of #744 AC-3 / #746 AC-4). Live run, three requests, recorded here.
- [ ] AC-3 `config-leaf-parity` and the SSOT lint stay green with the default change (parity updated if the census records it).

## Out of Scope

Retiring the `keep_alive` leaf or the `keepAlive` member (a configured value still has a lenient-server use); the Ollama provider; the readiness probe (#746, PR #748).

## Avoided Traps

Sniffing the host to decide what to send (the request shape must not depend on who answers). Keeping `-1` as the default and adding a hosted-only override through the overlay (the next hand-pointed strict endpoint breaks again). Reading LM Studio's 200 as "keep_alive works there" — it tolerates unknown fields; nothing documents a meaning.

## Related

#744 / PR #747 (the hosted lane) · #746 / PR #748 (the readiness probe) · #714 / PR #743 (the instrument that measures it) · `ai/provider/OpenAiCompatible.mjs` · neomjs/neo-agent-institution#351 (parent)

Decision Record impact: aligned-with ADR 0019 (a leaf default is a declared value; the provider reads it at the use site).

Sweeps: live latest-open sweep — the latest 12 open Brain issues at 2026-10-02T16:46Z (newest #766), no equivalent; search over all states for OpenAiCompatible + payload + unknown field returned #446 (schema union types, closed) only; A2A in-flight sweep — the inbox through 16:45Z, no claim on the client's wire shape; Memory Core rationale sweep — the 2026-09 provider-lane measurements ran LM Studio only; own-assignment sweep — #51, #53, #50, #351, none overlapping; structure map — one provider module + its spec, no new file.

Origin Session ID: 1efa16ff-bd83-41e5-87dc-4c186b03b451
Retrieval Hint: "OpenAiCompatible payload unknown field keep_alive operationStage Gemini 400 strict endpoint call-site options strip"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 1efa16ff-bd83-41e5-87dc-4c186b03b451


## Timeline

- 2026-10-02T16:49:18Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-02T16:49:20Z @neo-fable-clio added the `bug` label
- 2026-10-02T16:49:20Z @neo-fable-clio added the `ai` label
- 2026-10-02T16:49:21Z @neo-fable-clio added parent issue #351
- 2026-10-02T16:57:23Z @neo-fable-clio cross-referenced by #746
- 2026-10-02T16:57:24Z @neo-fable-clio cross-referenced by #744
- 2026-10-02T16:57:42Z @neo-fable-clio cross-referenced by PR #770
- 2026-10-02T17:09:37Z @neo-fable-clio referenced in commit `a4f77eb` - "fix(provider): the OpenAI-compatible client sends request fields only — the SSOT lint's comments name the decision record without a decaying ref (#767)"
- 2026-10-02T17:17:42Z @neo-fable-clio cross-referenced by #773
- 2026-10-02T17:28:55Z @tobiu referenced in commit `73a338c` - "fix(provider): the OpenAI-compatible client sends request fields only — strict endpoints accept the request (#767) (#770)

* fix(provider): the OpenAI-compatible client sends request fields only — call-site bookkeeping stripped, keep_alive only when configured, a null default projected as null (#767)

* fix(provider): the OpenAI-compatible client sends request fields only — the SSOT lint's comments name the decision record without a decaying ref (#767)"
- 2026-10-02T17:28:55Z @tobiu closed this issue
- 2026-10-02T17:32:51Z @neo-fable-clio cross-referenced by #776
- 2026-10-02T17:55:41Z @neo-gpt cross-referenced by PR #775

