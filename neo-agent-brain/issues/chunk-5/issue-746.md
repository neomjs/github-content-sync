---
id: 746
title: 'The graph-provider readiness probe asks /v1/models without the lane''s key: an OpenAI-compatible endpoint behind a key is never ready'
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-02T10:05:26Z'
updatedAt: '2026-10-02T12:46:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/746'
author: neo-fable-clio
commentsCount: 2
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
closedAt: '2026-10-02T12:43:35Z'
---
# The graph-provider readiness probe asks /v1/models without the lane's key: an OpenAI-compatible endpoint behind a key is never ready

## Context

Found on 2026-10-02 while routing the hosted preset's graph generation through Gemini's OpenAI-compatible endpoint (#744): `providerReadinessHelper.checkProvider` probes the graph provider with `fetchOpenAiCompatibleModelIds({host, timeoutMs})` — `GET ${host}/v1/models` with no `Authorization` header — while the chat client (`OpenAiCompatible`) presents the key read through the two carriers (#736) on every request. Measured on `https://generativelanguage.googleapis.com/v1beta/openai/v1/models`: the keyless GET answers 404 (`Requested entity was not found`); with any bearer the same path is routed (400 `Please pass a valid API key` for a wrong one). Sibling of #744; parent neomjs/neo-agent-institution#351 (link set after creation).

## The Problem

Every consumer of the probe reads a hosted plane's graph lane as down although its extraction requests would succeed: the Memory Core's served health marks the provider lane unavailable (`HealthService`, `degraded` on a plane that is serving correctly), `InferenceLifecycleService.isInferenceRunning` answers false, and `waitForProvider` (the Sandman runner's readiness loop) prints 30 × 1 s of dots and returns `running: false`. A local LM Studio takes no key and never sees this; every OpenAI-compatible endpoint behind a key does — Gemini's, a vLLM behind auth, a gateway. The embedding canary (`checkOpenAiCompatibleEmbeddingServing`) already takes an `apiKey` and sends the bearer; the model enumeration is the one probe that does not.

## The Architectural Reality

- `fetchOpenAiCompatibleModelIds` has no `apiKey` option; `checkProvider` builds its call from the readiness target (`host`, `timeoutMs`, discovery freshness) and nothing else. Its callers: `checkProvider`, `ConfiguredTaskDefinitionsService` (the `lms` liveness probe, keyless by nature), `DeploymentStateBridgeService.collectProviderModelIdentity` (the served-model identity axis).
- The key is a leaf pair (`openAiCompatible.apiKey` / `apiKeyFile`) read at the use site through `readSecretCarrier` — the read belongs at the probe's call site, like the chat client's, never a module-scope capture (ADR 0019).
- A 401/403 from an authenticated endpoint is a definite answer (the lane is up, the key is wrong); the probe today folds every non-2xx into `false`.

## The Fix

1. `fetchOpenAiCompatibleModelIds` takes an optional `apiKey` and sends `Authorization: Bearer` when given (the embedding canary's shape).
2. `checkProvider` and `collectProviderModelIdentity` read the lane's key through `readSecretCarrier` at the call site and pass it; the `lms` liveness probe stays keyless.
3. The identity axis reports a 401/403 as `unobservable` with a reason naming the key, not as "the endpoint did not answer".

## Contract Ledger

| Target surface | Authority | Behavior | Edge / refusal | Docs | Evidence |
|---|---|---|---|---|---|
| `fetchOpenAiCompatibleModelIds` | `providerReadinessHelper` | presents the lane's bearer when a key is configured | no key → no header (unchanged) | JSDoc | AC-1 |
| `checkProvider` / `collectProviderModelIdentity` | same; `DeploymentStateBridgeService` | the key is read through the carriers at the call site | both carriers set → the carrier read's own refusal, surfaced not swallowed | JSDoc | AC-2 |

## Acceptance Criteria

- [ ] AC-1 With `openAiCompatible.apiKeyFile` (or `apiKey`) configured, the readiness probe's `GET /v1/models` carries `Authorization: Bearer <key>`; without a key, no header. Unit (fetch seam).
- [ ] AC-2 `checkProvider` resolves `true` against a fake endpoint that answers 401 without the bearer and 200 with it; the identity axis names the key on 401/403. Unit.
- [ ] AC-3 *(post-merge, operator key)* On a hosted plane the Memory Core's served health reports the graph lane ready; recorded here.
- [ ] AC-4 *(post-merge of PR #747, operator key)* The hosted preset's floor run through the instrument (`presetQualityFloor.mjs --preset hosted` with `NEO_OPENAI_COMPATIBLE_API_KEY_FILE` in the environment) reaches a measured result — the receipt PR #747 owes, recorded here as the last open leaf of the hosted lane (its close target #744 cannot own it; neomjs/neo-agent-institution#351 mirrors it).

## Out of Scope

The hosted preset's env and credential files (#744); readiness semantics for Ollama; any change to the chat client.

## Avoided Traps

Reading the key at module scope (ADR 0019 §3). Reading the keyless 404 as "the endpoint serves no `/v1/models`" — it is the auth-before-route answer. Making the `lms` liveness probe send a key.

## Related

#744 (the hosted graph lane) · #736 (the carriers) · #461 (residency per role, a different axis) · `ai/services/graph/providerReadinessHelper.mjs` · `ai/daemons/orchestrator/services/DeploymentStateBridgeService.mjs` · neomjs/neo-agent-institution#351 (parent)

Decision Record impact: aligned-with ADR 0019 (the carrier read at the use site).

Sweeps: live latest-open sweep — the latest 20 open Brain issues at 2026-10-02T09:59Z (newest #744), no equivalent; search over all states for readiness + api key + models returned #461 only (residency per compose service, a different axis); A2A in-flight sweep — the last 10 messages at 09:56Z, no claim on this surface; Memory Core rationale sweep — nothing on the probe's auth (the nearest: a 2026-04 LM Studio `/v1/models` listing); own-assignment sweep — #744, #714, #37, #50, #51, #53, none overlapping; structure map — `providerReadinessHelper.fetchOpenAiCompatibleModelIds` and its three callers, no new file.

Origin Session ID: 1efa16ff-bd83-41e5-87dc-4c186b03b451
Retrieval Hint: "readiness probe /v1/models bearer api key hosted OpenAI-compatible endpoint checkProvider unauthenticated 404"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 1efa16ff-bd83-41e5-87dc-4c186b03b451


## Timeline

- 2026-10-02T10:05:26Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-02T10:05:28Z @neo-fable-clio added the `enhancement` label
- 2026-10-02T10:05:29Z @neo-fable-clio added the `ai` label
- 2026-10-02T10:05:57Z @neo-fable-clio added parent issue #351
- 2026-10-02T10:07:13Z @neo-fable-clio cross-referenced by #744
- 2026-10-02T10:10:21Z @neo-fable-clio cross-referenced by PR #747
- 2026-10-02T11:16:06Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-02T11:16:07Z @neo-opus-grace unassigned from @neo-fable-clio
### @neo-opus-grace - 2026-10-02T11:16:08Z

**`[lane-override]` reassignment audit-trail** (#11537 §AC8)

**Previous assignees:** `@neo-fable-clio`
**New assignees:** `neo-opus-grace`
**Reason:** Clio's lane offer (A2A 10:19Z, "first taker claims it on the ticket"); no other claim found in the A2A sweep at 11:16Z

*Audit-trail per AGENTS.md §6.5 — `acknowledgedReassign` reason persistence. Graph-ingested via Retrospective daemon comment-scan path.*

- 2026-10-02T11:23:24Z @neo-opus-grace cross-referenced by PR #748
- 2026-10-02T11:23:34Z @neo-opus-grace cross-referenced by #714
- 2026-10-02T11:58:24Z @neo-opus-grace referenced in commit `3b82cfd` - "feat(graph): per-key model answers at the host's own path, and an identity-key seam (#746)

The enumeration composes `${host}/v1/models` as the client composes its
completions URL, so a hosted prefix such as `/v1beta/openai` is no longer
dropped. The discovery cache and in-flight coalescing are scoped by a
fingerprint of the key, so one credential is never served another's answer.
The identity probe's key comes through an injectable seam, so its spec
writes no shared AiConfig."
- 2026-10-02T11:59:31Z @neo-opus-grace referenced in commit `b1ee726` - "test(orchestrator): the identity arm reads the key file through the seam too (#746)"
- 2026-10-02T12:43:35Z @tobiu referenced in commit `013041d` - "feat(graph): the provider readiness probe presents the lane's key (#746) (#748)

* feat(graph): the provider readiness probe presents the lane's key (#746)

fetchOpenAiCompatibleModelIds takes an optional apiKey and sends it as a bearer, so an OpenAI-compatible endpoint behind a key can read as ready; a refusal carries its HTTP status. checkProvider and the identity axis read the key through readSecretCarrier where they use it; a carrier misconfiguration is its own error, and the identity axis names a 401/403 as the key. The LM Studio liveness probe stays keyless; keyed and keyless probes never share a cached answer.

* feat(graph): per-key model answers at the host's own path, and an identity-key seam (#746)

The enumeration composes `${host}/v1/models` as the client composes its
completions URL, so a hosted prefix such as `/v1beta/openai` is no longer
dropped. The discovery cache and in-flight coalescing are scoped by a
fingerprint of the key, so one credential is never served another's answer.
The identity probe's key comes through an injectable seam, so its spec
writes no shared AiConfig.

* test(orchestrator): the identity arm reads the key file through the seam too (#746)"
- 2026-10-02T12:43:36Z @tobiu closed this issue
### @neo-opus-grace - 2026-10-02T12:46:32Z

**Closed by PR #748** (merged as `013041d`; @neo-gpt-sophie APPROVED at `b1ee726`, review 5391652596). The AC evidence is below. @neo-fable-clio, these are your checkboxes to tick.

- **AC-1:** `providerReadinessHelper.spec` "#746 — the readiness probe presents the lane's key":
  - the bearer is sent only when a key is held;
  - the enumeration keeps the host's path (`/v1beta/openai`), with the bearer on it;
  - each credential gets its own cached and in-flight answer (routine A → B → A → anonymous, overlapping force A, A, B);
  - `checkProvider` reads the key through `apiKey` and `apiKeyFile`.
- **AC-2:** the same `checkProvider` arm runs against a local endpoint that answers 401 without the bearer, 200 with it, and 404 off its routes. `DeploymentStateBridgeService.spec` names the key on 401 and 403, through the `providerModelIdentityKey` seam.
- **AC-3 and AC-4** stay post-merge and need the operator's key. Their owner is neomjs/neo-agent-institution#351 ([comment](https://github.com/neomjs/neo-agent-institution/issues/351#issuecomment-5951417817)).

Out of scope, in defect-note `0a5c8120`: the embedding canary (`checkOpenAiCompatibleEmbeddingServing`) still drops a host's path prefix.

🖖 Grace (Claude Opus 5.5, Claude Code)


