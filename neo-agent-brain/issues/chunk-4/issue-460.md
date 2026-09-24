---
id: 460
title: A wrong-shape LM Studio resident blocks loading every missing role
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-24T14:31:39Z'
updatedAt: '2026-09-24T16:19:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/460'
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
closedAt: '2026-09-24T16:19:26Z'
---
# A wrong-shape LM Studio resident blocks loading every missing role

## Context

The operator set the requirement on 2026-09-24: the chat and embedding models stay resident, and never unload, on every provider Agent OS runs against. neo#17079 already states the local contract. Routine readiness may add a genuinely missing model and never evicts. Replacing a wrong-shape resident is an explicit operator action.

That day on the local plane, LM Studio logged `Failed to load model "google/gemma-4-26b-a4b". Error: Operation canceled` 126 times, before and after the 13:28Z plane cut, against 0 on 09-22 and 09-23. Session summaries, the dream's LLM stage and KB synthesis failed with it. LM Studio's server log records CEST local time; the times below are converted to UTC.

## The Problem

Measured on 2026-09-24:
- **The embedding model was resident as a JIT instance at the wrong context.** `lms ps --json` showed `text-embedding-qwen3-embedding-8b` at `contextLength 8192` with `ttlMs 3600000`. A consumer request had loaded it at LM Studio's default shape, and the configured shape is 32768.
- **host-edge readiness loaded nothing because of it.** A dry run of `ensureLmsModelsLoaded` with host-edge's own environment returned `observationStatus: replacement-required` for the embedding model, `missingModels: []`, and no load attempt. The chat model was not resident at all. LM Studio's log shows no `lms-cli loadModel` for either model all day.
- **The chat model was therefore only ever loaded just-in-time**, and with LM Studio's `unloadPreviousJITModelOnLoad: true` the next embedding request cancelled it. At 11:51:37Z a chat completion started the load; at 11:51:41Z an embeddings request arrived and the load was cancelled.
- **Operational mitigation applied (operator-authorized, 2026-09-24):** both models were pinned at their configured shapes at 14:27Z, and host-edge readiness has been green since. LM Studio's `justInTimeModelLoading` and `unloadPreviousJITModelOnLoad` were switched off at 14:40Z. Verified: after a server restart both models were still pinned, and a chat request for the unloaded `gpt-oss-20b` loaded nothing; LM Studio answered with the resident model, which is its behavior with JIT off. On this plane a model can now become resident only through an explicit load. This ticket's code defect remains for any plane where JIT is on.
- **Nobody saw the report.** The supervisor printed `readiness hook completed with degraded readiness after liveness confirmation.` every 15 s with no summary after it, and the helper's own `log.warn` does not reach host-edge's log files.
- **Pinning fixed it.** Loading both models by hand at their configured shapes (`lms load … --context-length … --parallel 1`, no `--ttl`) made the hook report success at 14:27:33Z, the first success of the day.

## The Architectural Reality

- `providerReadinessHelper.mjs:2056`: `initialRefusal = rejectNonAuthorizingObservation(assessment, …)` returns before the per-model loop (`:2095–2150`) that performs additive loads. One role's `replacement-required` result therefore refuses every other role's load. That contradicts neo#17079's "routine readiness may add a genuinely missing model".
- The refusal carries a `warning` ("LM Studio loaded-model replacement requires an explicit operator action: …"). `ProcessSupervisorService.getReadinessOperatorSummary` (`:501`) reads only `result.operatorDiagnostic.summary`, so the warning never reaches the line an operator reads.
- A transient instance can be told from a pinned one: `lms ps --json` reports `ttlMs` as a positive number for a JIT load and `null` for an `lms load` without `--ttl`.

## The Fix

1. Assess residency per role. A role whose resident has the wrong shape stays report-only, as neo#17079 requires. Every genuinely missing role still gets its additive, shaped, TTL-free load in the same pass, and the result stays degraded while any role needs replacement.
2. The `replacement-required` result carries `operatorDiagnostic.summary`. It names the role, the model, the loaded and required context and parallel, whether the resident is transient (`ttlMs`) or pinned, and the `lms unload …` / `lms load … --context-length … --parallel …` pair that performs the replacement. The supervisor's degraded line prints it.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `ensureLmsModelsLoaded` result | neo#17079, `providerReadinessHelper.mjs` | missing roles load in the same pass as a `replacement-required` role; the result stays `degraded` | unknown metadata still refuses every mutation (`metadata-unknown`), unchanged | JSDoc on `ensureLmsModelsLoaded` (the report-only rule gains the per-role clause) | helper spec: one role mis-shaped, one missing → one load, one report |
| `operatorDiagnostic.summary` for `replacement-required` | `ProcessSupervisorService.getReadinessOperatorSummary` | a summary naming the role, model, shapes, transient or pinned, and the replacement commands | none; it is always present on this status | JSDoc on the result shape | supervisor spec: the degraded line carries the summary |

## Acceptance Criteria

- [ ] **AC-1** Role A resident at the wrong shape and role B missing: one readiness pass loads B (additive, shaped, no TTL) and reports A as `replacement-required` without mutating it. A mutation control that restores the early return turns the arm red.
- [ ] **AC-2** The supervisor's degraded readiness line names the role, the model, the loaded and required shape, transient or pinned, and the replacement commands.
- [ ] **AC-3** *(deployed plane, `[L4-deferred — operator handoff needed]`)* After an LM Studio restart that leaves no model resident, host-edge pins both roles at their configured shapes with no operator action (`lms ps --json`: `ttlMs: null`, contexts ≥ 131072 and 32768), and the readiness line reports success. Recorded on this ticket. The mis-shaped-resident arm cannot recur on this plane now that JIT is off, so AC-1's unit fixture carries it.

## Out of Scope

- Automatically replacing a wrong-shape resident, even a transient one. neo#17079 removed automatic eviction after it ejected configured roles (neo#17051, neo#17071), and reopening that needs its own decision.
- LM Studio's JIT and auto-evict settings. They are operator-owned app settings, switched off on this plane under the operator's decision; other planes keep their own choice, so the code fix still applies there.
- A cross-provider role-residency readout in the plane healthcheck (Ollama, MLX, remote OpenAI-compatible). This is the provider-agnostic follow-up the operator asked for, filed after this.
- Identifier collisions on load (#29).

## Avoided Traps

- **Normalizing shape by eviction.** That is neo#17079's rejected authority.
- **Treating an LM Studio flag as the fix.** The flag changes when a JIT load fires, not what readiness does with the result.

## Related

neo#17079 · neo#13851 (wrong-context resident starved the golden path) · neo#17051 · neo#17071 · #29 · #47 · #23

Live latest-open sweep: the latest 20 open `neomjs/neo-agent-brain` issues at 2026-09-24T14:30Z; none equivalent. Exact search (`residency`, `LM Studio`, `replacement-required`, `JIT`) across both repositories: neo#17079 and its siblings (closed, the contract this restores), plus #29 (open, identifier collisions, different). A2A in-flight sweep (the 30 most recent): no claim, and @neo-fable-clio's 14:02Z defect-note is this symptom. MC sweep: the June "role-set residency repair with hysteresis" analysis and my July finding that an explicit `lms load` carries no TTL; no conflicting decision. Own-assignment sweep: #23 (embedding lane geometry) is adjacent, not equivalent. Structure map: both files already sit under their owners (`ai/services/graph/`, `ai/daemons/orchestrator/services/`).

Origin Session ID: 9f7b8241-8b3c-4954-a9e5-2f9c1e41d669
Retrieval Hint: `query_raw_memories("LM Studio replacement-required blocks additive load missing role JIT embedding context 8192 gemma Operation canceled")`

Authored by Vega (Opus 5.5, Claude Code) 🌿



## Timeline

- 2026-09-24T14:31:39Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-24T14:31:40Z @neo-opus-vega added the `bug` label
- 2026-09-24T14:31:40Z @neo-opus-vega added the `ai` label
- 2026-09-24T14:31:41Z @neo-opus-vega added the `agent-os` label
- 2026-09-24T14:34:29Z @neo-opus-vega cross-referenced by #461
- 2026-09-24T15:11:22Z @neo-opus-vega cross-referenced by PR #462
- 2026-09-24T15:52:47Z @neo-opus-vega referenced in commit `ad7048a` - "fix(lms): an absent or unusable ttlMs reads as TTL unknown, never as pinned (#460)

describeLmsReplacement treated every value that was not a positive number as pinned, so a row without ttlMs (an older lms, or a field the normalizer did not copy) claimed the very residency guarantee the operator reads the line to check. Now only ttlMs: null, which an explicit lms load reports, says pinned; a positive number is a JIT load; anything else is TTL unknown. A new arm covers an absent and an untyped value, and restoring the old fallthrough turns it red."
- 2026-09-24T16:19:26Z @tobiu referenced in commit `6057492` - "Merge pull request #462 from neomjs/vega/460-lms-per-role-residency

fix(lms): a mis-shaped LM Studio resident no longer blocks loading the missing roles (#460)"
- 2026-09-24T16:19:26Z @tobiu closed this issue

