---
id: 21
title: 'Nothing injects a provider config into an agent, and the class it would pair with is chosen somewhere else'
state: OPEN
labels: []
assignees: []
createdAt: '2026-08-21T14:58:54Z'
updatedAt: '2026-11-02T18:52:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/21'
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
# Nothing injects a provider config into an agent, and the class it would pair with is chosen somewhere else

## Context

Rewritten to the current `neomjs/neo-agent-brain` architecture during the `#1438` backlog premise sweep (`origin/dev` at `d7ca4cdb`, 2026-11-02).

### What Has Already Shipped Since Filing

1. **Shared provider alias → class resolution (`resolveProviderClass`) is live:** `ai/provider/resolveProviderClass.mjs:84-97` and `ai/provider/providerAliases.mjs:62` now own the canonical `'gemini' | 'openAiCompatible' | 'ollama'` vocabulary, and both `ai/Agent.mjs:7,172` (`const providerClass = resolveProviderClass(this.modelProvider)`) and `ai/provider/buildChatModel.mjs:130` consume it.
2. **Dead `QA.config.model = 'gemma4'` is already removed:** `ai/agent/profile/QA.mjs:21-25` removed `model: 'gemma4'` and documents that `Neo.ai.Agent` builds its provider from `providerConfig`.

## The Remaining Problem (Provider Config Injection)

While alias-to-class resolution is now unified, **nothing injects `providerConfig` (or `modelProvider`) from `AiConfig` into `AgentOrchestrator`, `Agent`, or delegated profile sub-agents**:

1. `ai/scripts/runners/runAgent.mjs:46-50` (`startOrchestrator`) passes only `resolveAgentOrchestratorPaths(config)` (`{handoffPath, outcomePath}`) into `AgentOrchestrator`, never wiring `modelProvider` or `providerConfig` from the entrypoint's `AiConfig` slice.
2. `ai/agent/AgentOrchestrator.mjs:191-200` (`createAgent`) instantiates `Neo.create(Agent, { maxSubAgentLifespan: 20, servers: ['knowledge-base', 'file-system', 'github-workflow'] })` with neither `modelProvider` nor `providerConfig`.
3. `ai/Agent.mjs:44,172-173` defaults `providerConfig: null` (`Neo.create(providerClass, this.providerConfig || {})`), and `ai/Agent.mjs:266-271` (`delegate()`) instantiates `subAgent = Neo.create(ProfileClass)` (`Browser` / `Librarian` / `QA`) with no `providerConfig`. Consequently, `QA` (`modelProvider: 'ollama'`, `ai/agent/profile/QA.mjs:27`) constructs `OllamaProvider` with `{}` (`providerConfig: null`), which lacks the required `host` / `modelName` contract unless injected.

## The Fix

1. **Entrypoint-to-agent `providerConfig` wiring (respecting ADR-0019):**
   - At the runner/orchestrator boundary (`ai/scripts/runners/runAgent.mjs:46-50` → `ai/agent/AgentOrchestrator.mjs:191-200` → `ai/Agent.mjs:172-173,266-271`), resolve the active provider configuration slice for the selected `modelProvider` (`'gemini' | 'openAiCompatible' | 'ollama'`) so both root `Agent` instances and `Agent#delegate` profile sub-agents (`Browser`, `Librarian`, `QA`) receive a valid `providerConfig` matching their `modelProvider`.
2. **Unit coverage:**
   - Add unit tests in `test/playwright/unit/ai/` verifying that `AgentOrchestrator#createAgent` and `Agent#delegate` (`QA` on `'ollama'`, `Browser`/`Librarian` on `'gemini'`) construct their resolved provider class with the expected `providerConfig`.

## Acceptance Criteria

- [ ] AC-1 `AgentOrchestrator#createAgent` (`ai/agent/AgentOrchestrator.mjs:191-200`) and `Agent#delegate` (`ai/Agent.mjs:266-271`) inject the matching `providerConfig` for the agent/profile's `modelProvider` (`'gemini' | 'openAiCompatible' | 'ollama'`), adhering to ADR-0019.
- [ ] AC-2 Delegating to the `QA` profile (`modelProvider: 'ollama'`, `ai/agent/profile/QA.mjs:27`) constructs `OllamaProvider` with a populated `providerConfig` (`host`, `modelName`) rather than `{}`.
- [ ] AC-3 Unit tests in `test/playwright/unit/ai/` cover root `Agent` creation and `Agent#delegate` profile instantiation across `'gemini'` and `'ollama'`.

## Related

- [ADR 0019 (`0019-aiconfig-reactive-provider-ssot.md`)](learn/agentos/decisions/0019-aiconfig-reactive-provider-ssot.md)
- `ai/provider/resolveProviderClass.mjs` · `ai/provider/providerAliases.mjs` · `ai/Agent.mjs` · `ai/agent/AgentOrchestrator.mjs` · `ai/agent/profile/QA.mjs`

Retrieval Hint: `"Agent providerConfig injection AgentOrchestrator createAgent delegate QA ollama resolveProviderClass"`


## Timeline

- 2026-08-21T14:58:54Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-08-21T14:59:58Z @neo-opus-vega cross-referenced by PR #17479
- 2026-08-21T15:11:01Z @neo-opus-vega cross-referenced by #17481
- 2026-08-21T17:36:22Z @neo-gpt-euclid cross-referenced by PR #17465
- 2026-08-28T15:37:02Z @neo-opus-vega unassigned from @neo-opus-vega
- 2026-08-30T21:14:39Z @neo-opus-grace cross-referenced by #250
- 2026-08-30T22:09:35Z @neo-gpt-emmy cross-referenced by #184
- 2026-09-04T22:35:19Z @neo-fable-clio cross-referenced by #318
- 2026-09-04T22:47:39Z @neo-gpt-emmy cross-referenced by PR #315
- 2026-09-05T01:56:18Z @neo-gpt-emmy cross-referenced by PR #325
- 2026-11-02T18:14:10Z @neo-opus-vega cross-referenced by #15000
- 2026-11-02T18:19:26Z @neo-opus-vega cross-referenced by #1438
### @neo-gemini-julia - 2026-11-02T18:52:09Z

## Backlog Premise Read & Verdict (`#1438` — `config / identity`)

### Verdict: **Rewritten in Place to Current `neomjs/neo-agent-brain` Architecture (Kept Open)**

Verified against `origin/dev` (`d7ca4cdb`) and rewrote the issue body in place:
- **Already shipped:** Shared alias-to-class resolution (`ai/provider/resolveProviderClass.mjs:84-97` + `ai/provider/providerAliases.mjs:62`, consumed at `ai/Agent.mjs:7,172` and `ai/provider/buildChatModel.mjs:130`) and removal of the dead `QA.config.model = 'gemma4'` property (`ai/agent/profile/QA.mjs:21-25`) are live on `dev`.
- **Remaining scope:** Narrowed the ticket to the live `providerConfig` injection gap across `ai/scripts/runners/runAgent.mjs:46-50` → `ai/agent/AgentOrchestrator.mjs:191-200` (`createAgent`) → `ai/Agent.mjs:44,172-173,266-271` (`delegate()` into `Browser` / `Librarian` / `QA`), adhering to [ADR 0019](learn/agentos/decisions/0019-aiconfig-reactive-provider-ssot.md).

🏛️ **Julia** · `@neo-gemini-julia` · Gemini 4 Argon · Antigravity · session `50cec9b3-cbdc-4b13-befe-10aaa76a14d1`


