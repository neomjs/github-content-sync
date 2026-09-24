---
id: 461
title: 'Residency is observed per compose service, never per model role'
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-09-24T14:34:28Z'
updatedAt: '2026-09-24T16:48:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/461'
author: neo-opus-vega
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
# Residency is observed per compose service, never per model role

## Context

The operator's requirement from 2026-09-24: the chat and embedding models stay resident on every provider Agent OS runs against. That means LM Studio on a workstation and whatever a real cloud deployment uses. #460 fixes the LM Studio repair. This ticket makes residency observable on every provider, so no plane can run for a day without its chat model and look healthy. The local plane did exactly that on 2026-09-24.

## The Problem

- `DeploymentStateBridgeService.collectProviderResidency` runs only for the services listed in `orchestrator.deploymentStateBridge.providerResidencyServiceKeys`. The default is `['local-model', 'model']` (`configBase.mjs:1583`), which means a model container inside the compose project.
- On the local plane both roles are served by LM Studio on the host (`host.docker.internal:1234`), so no service is eligible. At 14:3xZ on 2026-09-24, `deployment-state/snapshot.json` carried `providerResidency: null` and `providerResidencyEligible: false` for all five services.
- The MC healthcheck's `providers` block names each role's model but never says whether it is loaded. It read the same all day while the chat model was absent (#460).
- A remote OpenAI-compatible or hosted provider in a cloud deployment goes unobserved in the same way.
- Each provider already exposes the facts:
  - LM Studio: `/api/v0/models` (`state`, `loaded_context_length`) and `lms ps --json` (`ttlMs`: positive means transient/JIT, `null` means pinned).
  - Ollama: `/api/ps` (loaded models and `expires_at` under `keep_alive`).
  - A remote endpoint: availability through `/v1/models`, but not residency.

## The Architectural Reality

- Residency repair already exists per provider. LM Studio uses host-edge `ensureLmsModelsLoaded` (#460). Ollama in the cloud compose runs with `OLLAMA_KEEP_ALIVE=-1` and `OLLAMA_MAX_LOADED_MODELS=2`, and the provider sends `keep_alive: -1` on chat and embedding requests (`ai/provider/Ollama.mjs:393`, `:541–559`).
- What is missing is the observation, keyed by role rather than by compose service.
- The role → provider → endpoint map already lives in config: `modelProvider`, `graphProvider`, `embeddingProvider` and each provider's host and model leaves. No new leaf is expected. Any config touch reads ADR-0019 first (critical gate 10).

## The Fix

Observe residency per role (chat, embedding) against that role's configured provider endpoint, with one adapter per provider kind:
- LM Studio: loaded, shape, and pinned versus TTL.
- Ollama: loaded, and expiry under `keep_alive`.
- Remote or hosted: `external`, meaning availability only.

The existing compose-service observation remains the in-plane adapter's input, not the gate. The snapshot and the MC healthcheck carry the per-role record `{role, provider, model, state: resident | transient | missing | mis-shaped | external | unknown, contextLength, requiredContextLength, observedAt}`. A `missing` or `mis-shaped` role degrades the healthcheck and names the role.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| per-role residency in `deployment-state/snapshot.json` | `DeploymentStateBridgeService` | one record per configured role, from that role's provider | `unknown` with the probe error when the endpoint does not answer; `external` for providers whose residency is not observable | JSDoc on the collector and the record shape | bridge spec: fixtures per adapter |
| MC healthcheck `providers.<role>.residency` | `HealthService` | shows the record; `missing` or `mis-shaped` degrades status and names the role | absent record → `unknown`, never `resident` | healthcheck JSDoc; the operator runbook line that reads it | health spec: degrade arm and resident arm |

## Acceptance Criteria

- [ ] **AC-1** For each role, the snapshot records residency from its configured provider. Unit fixtures cover LM Studio (pinned, transient, missing, mis-shaped), Ollama (loaded, expired) and a remote OpenAI-compatible host (`external`).
- [ ] **AC-2** The MC healthcheck shows per-role residency, and when a role is `missing` or `mis-shaped` it degrades and names that role.
- [ ] **AC-3** *(deployed plane, `[L4-deferred — operator handoff needed]`)* On the local plane, the healthcheck shows both roles resident and pinned. Recorded on this ticket.
- [x] **AC-4** *(deployed plane, carried as residual owner for #460 / PR #462)* After an LM Studio restart that leaves no model resident, host-edge pins both roles at their configured shapes with no operator action (`lms ps --json`: `ttlMs: null`), and its readiness line reports success. Recorded on this ticket. Receipt: comment 5818358296 (unload at 16:47:08Z, both pinned by 16:47:30Z, at Brain `6057492`).

## Out of Scope

- Repairing residency. That belongs to #460 for LM Studio; Ollama is already configured to keep both models loaded.
- Automatically replacing a wrong-shape resident (neo#17079).
- The embedding lane's parallelism and geometry authority (#23).

## Avoided Traps

- **Reading `/v1/models` as residency.** LM Studio lists every downloaded model there, loaded or not.
- **Keying the observation by compose service.** The role's endpoint can be a container, a host app or a remote API, and only the role is constant across deployments.

## Related

#460 · neo#17079 · neo#13942 (active-provider recovery) · #23 · #47

Live latest-open sweep: the latest 20 open `neomjs/neo-agent-brain` issues at 2026-09-24T14:3xZ, plus #460 filed just before; none equivalent. A2A in-flight sweep (the 30 most recent, 14:30Z): no claim. MC sweep: the June active-provider recovery daemon and the "role-set residency repair" analysis, both repair-side; no observation-side decision. Own-assignment sweep: #23 is adjacent, not equivalent. Structure map: `ai/daemons/orchestrator/services/DeploymentStateBridgeService.mjs` and `ai/services/memory-core/HealthService.mjs` are existing owners.

unowned-rationale: filed by #460's author and open to any seat. It is larger than a cleanup lane, and the author takes it if it is unclaimed once #460 lands.

Origin Session ID: 9f7b8241-8b3c-4954-a9e5-2f9c1e41d669
Retrieval Hint: `query_raw_memories("provider residency per role healthcheck providerResidencyServiceKeys local-model LM Studio host Ollama remote external")`

Authored by Vega (Opus 5.5, Claude Code) 🌿




## Timeline

- 2026-09-24T14:34:30Z @neo-opus-vega added the `enhancement` label
- 2026-09-24T14:34:31Z @neo-opus-vega added the `ai` label
- 2026-09-24T14:34:31Z @neo-opus-vega added the `agent-os` label
- 2026-09-24T15:11:22Z @neo-opus-vega cross-referenced by PR #462
### @neo-opus-vega - 2026-09-24T16:48:36Z

**AC-4 receipt (#460's deployed check, carried here) on the local plane at Brain `6057492`, recreated 16:46Z on 2026-09-24.**

- **16:47:08Z:** `lms unload --all` left 0 models resident.
- **16:47:20Z:** host-edge's readiness pass issued `lms-cli loadModel` for `google/gemma-4-26b-a4b`.
- **16:47:28Z:** the same for `text-embedding-qwen3-embedding-8b`. Both calls are in LM Studio's own log; its times are CEST and are converted here.
- **16:47:30Z:** `lms ps --json` showed both pinned (`ttlMs: null`) at their configured shapes. gemma read `262144`, which is ≥ the required 131072 and classed sufficient. The embedder read `32768`.

Nobody acted between the unload and the pins, and LM Studio logged no `Operation canceled`. JIT is off on this plane, so no consumer request could race a wrong-shape instance in first. host-edge printed no degraded readiness line during the re-pin. Its last readiness line is the 16:46:19Z success after the recreate, because the supervisor logs success only on a state transition.

— Vega (Opus 5.5, Claude Code) 🌿


