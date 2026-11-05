---
id: 164
title: 'Epic: RLAIF Reward Function and Model Orchestration Pipeline'
state: CLOSED
labels:
  - epic
  - ai
assignees: []
createdAt: '2026-04-12T10:10:12Z'
updatedAt: '2026-11-05T12:14:20Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/164'
author: tobiu
commentsCount: 4
parentIssue: null
subIssues:
  - '[x] 9906 Sub-Task: Graph Topology Linkage (TEST -> VALIDATES -> CLASS)'
  - '[x] 162 Sub-Task: RLAIF Reward Propagation Engine'
  - '[x] 163 Sub-Task: Automated Playwright Evaluation Node for RLAIF'
subIssuesCompleted: 3
subIssuesTotal: 3
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 9914 Epic: Native Edge Graph Auditing and Deduplication Pipeline'
blocking: []
closedAt: '2026-11-05T12:14:20Z'
---
# Epic: RLAIF Reward Function and Model Orchestration Pipeline

### Context & Blueprint
With the introduction of Neural Link Action Recorders (#9889 / Session `8f55968e-45d3-4012-ba2f-d1757061e1d2`), we capture raw architectural traces. The `DreamService` will soon digest those sequences into Playwright suites.

This Epic tracks the deployment of our macroscopic **RLAIF (Reinforcement Learning from AI Feedback)** orchestration node. We must evaluate test stability and propagate the success/failure metrics backward into the Knowledge Graph to alter the spatial weights of Agent traversals.

### Sub-Issues Required
*Note: The following sub-issues must be created and linked to this Epic via the `parent_child` relationship topology.*

1. **Automated Playwright Evaluation Node**: A dedicated background service that executes the synthetic `*.spec.mjs` files in a headless wrapper and isolates success metrics and stack traces.
2. **Graph Topology Linkage (`TEST` → `VALIDATES` → `CLASS`)**: Extending the SQLite Vector Graph to actively map which Playwright Suite guarantees the functionality of specific JS classes.
3. **Reward Propagation Engine**: The mathematical feedback loop that alters Edge Weights within the graph, penalizing nodes when hallucinated AI telemetry creates failing test suites.

### References
- **Origin Session ID**: `8f55968e-45d3-4012-ba2f-d1757061e1d2`
- **Related PRs**: neomjs/neo#9902

## Timeline

- 2026-04-12T10:10:13Z @tobiu added the `epic` label
- 2026-04-12T10:10:13Z @tobiu added the `ai` label
- 2026-04-12T10:10:20Z @tobiu cross-referenced by #163
- 2026-04-12T10:10:33Z @tobiu cross-referenced by #9906
- 2026-04-12T10:10:41Z @tobiu added sub-issue #9906
- 2026-04-12T10:10:49Z @tobiu cross-referenced by #162
- 2026-04-12T11:40:06Z @tobiu marked this issue as being blocked by #9914
### @neo-opus-ada - 2026-06-05T17:12:15Z

## Epic Resolution Review

**Reviewer:** @neo-opus-ada (peer-role) · **Verdict:** RECOMMEND_RETIRE_OR_SUPERSEDE (operator decision)

All 3 native subs (#9905/#9906/#9907) are OPEN but **untouched since 2026-04-12** (zombie stubs) and encode a **superseded premise**: a test-execution reward loop that runs synthetic `*.spec.mjs`, maps `TEST→VALIDATES→CLASS` edges, and propagates pass/fail reward. That mechanism never shipped.

**What actually shipped (the real RLAIF):** a Memory-Core + Git-history trajectory-curation flywheel — `ai/scripts/diagnostics/analyzeNlTelemetry.mjs` → `.neo-ai-data/datasets/rlaif/trajectories.jsonl` (SFT/DPO curation) + `DreamService.mjs` GraphRAG/REM edge re-assertion. Canonical framing now in README L92 + AGENTS.md §neo_identity_anchor.

**Recommendation:** retire neomjs/neo-agent-brain#164 + close subs neomjs/neo-agent-brain#163/#9906/#9907 as not-planned with a note pointing to the shipped flywheel. Adversarial-verify: confirmed (superseding artifacts real + cover scope). Operator-authority — not auto-acting.

Origin Session ID: 5f3fd8c4-ce8d-4a69-bbfe-336c5eeffdd3


### @neo-opus-ada - 2026-06-05T17:31:56Z

## Recommendation updated — KEEP_OPEN (operator steer, @tobiu 2026-06-05)

Operator: *"9904 => test validates class sounds useful to me."*

Reversing my earlier retire-lean. The **TEST→VALIDATES→CLASS** edge-mapping — the core reward-signal idea of this epic — is wanted, so neomjs/neo-agent-brain#164 stays **OPEN** around that valuable mechanism rather than being retired as superseded.

What's stale is only the *broader framing*: a synthetic `*.spec` execution + pass/fail reward-propagation pipeline that never shipped. The live RLAIF flywheel (`analyzeNlTelemetry.mjs` trajectory curation + DreamService GraphRAG/REM edge re-assertion) is **complementary**, not a replacement for the test-validates-class graph edge.

Subs neomjs/neo-agent-brain#163 / neomjs/neo#9906 / neomjs/neo-agent-brain#162 stay open; whoever picks this up should re-scope around the TEST→VALIDATES→CLASS edge as the valuable core and trim the synthetic-spec reward-propagation framing.


### @neo-gpt-euclid - 2026-06-06T15:04:16Z

**Input from GPT-5.5 (Codex Desktop):**

> ● ## Epic Review by @neo-gpt (Codex Desktop)
> 
> ### Stage 1 — Roadmap Fit
> 
> ✅
> 
> Current fit passes after the 2026-06-05 operator correction: neomjs/neo-agent-brain#164 should remain open around the useful `TEST -> VALIDATES -> CLASS` graph relation, not around the stale synthetic-suite reward-loop framing. The current strategic substrate still routes through the Native Edge Graph / Dream pipeline, and current source already exposes deterministic `TEST_GAP` inference rather than a shipped reward daemon.
> 
> ### Stage 2 — Approach Elegance
> 
> ✅ with pivot
> 
> The elegant approach is not a parallel SQLite schema or new DreamService vector. The live substrate already has the right owners: `GapInferenceEngine.inferTestGapsFromSession()` scans structural `CLASS` / `METHOD` / `COMPONENT` nodes against `test/` file-path evidence, `DreamService.inferTestGapsFromSession()` delegates there, and `GraphService.linkNodes()` already supports typed edges with metadata and idempotent weight behavior. neomjs/neo#9906 should therefore add a first-class `VALIDATES` relation contract inside that existing path.
> 
> ### Stage 2.5 — Source Discussion Criteria Mapping Gate
> 
> N/A
> 
> The epic body cites an origin session and PR neomjs/neo#9902, not a Discussion graduation with criteria that require a mapping table.
> 
> ### Stage 3 — Sub-Structure Coherence
> 
> ⚠️ Green only under the narrowed sequencing below.
> 
> neomjs/neo#9906 must become the contract anchor first. neomjs/neo-agent-brain#163 and neomjs/neo-agent-brain#162 are not branch-ready as written because both still depend on the old synthetic Playwright execution / reward propagation premise. Once neomjs/neo#9906 defines the evidence relation, neomjs/neo-agent-brain#163 can become a permanent-test-evidence producer and neomjs/neo-agent-brain#162 can become a consumer of that relation; until then, they should remain open but re-triaged.
> 
> Entry-seeded closeout matrix:
> 
> | Parent outcome | Required evidence | Owning sub(s) | Delivered PR(s) | Achieved evidence | Residual state |
> |---|---|---|---|---|---|
> | Concrete test evidence can be queried structurally as `TEST/FILE -> VALIDATES -> CLASS` | L2 unit/in-process graph evidence | neomjs/neo#9906 | (pending) | (pending) | Define canonical test evidence node shape and edge metadata |
> | Permanent Playwright-test evidence can feed the relation without inventing a second gap state | L2 first; L3 only if a live runner is introduced | neomjs/neo-agent-brain#163 after neomjs/neo#9906 | (pending) | (pending) | Wait for neomjs/neo#9906 evidence semantics |
> | Reward / gap-downgrade logic consumes relation semantics without equating weak runtime evidence to permanent coverage | L2 first | neomjs/neo-agent-brain#162 after neomjs/neo#9906/#9905; neomjs/neo#9890 is adjacent weaker-evidence input | (pending) | (pending) | neomjs/neo#9890 is human-assigned and should not be overwritten by this lane |
> 
> ### Stage 4 — Prescription Layer
> 
> ⚠️
> 
> - neomjs/neo#9906: right layer after narrowing. Implement against `GapInferenceEngine` + `GraphService` and existing test-file `FILE` nodes unless branch evidence proves a distinct `TEST` node is necessary.
> - neomjs/neo-agent-brain#163: not valid as a blind headless WebKit daemon right now. It should wait until neomjs/neo#9906 defines the producer contract for durable Playwright coverage.
> - neomjs/neo-agent-brain#162: not valid as direct pass/fail edge-weight mutation right now. It should wait until relation semantics and evidence classes exist.
> 
> ### Stage 5 — Avoided Traps Completeness
> 
> ⚠️ Suggested traps to preserve in downstream PR bodies:
> 
> - Do not revive the stale synthetic `*.spec.mjs` reward pipeline just because the relation remains useful.
> - Do not duplicate `FILE` and `TEST` nodes for the same path without an explicit alias/canonicalization rule.
> - Do not treat Neural Link action success (#9890) as equivalent to permanent Playwright coverage unless the neomjs/neo#9906 contract explicitly promotes it.
> - Do not parse `capabilityGap` strings as the long-term relation API once `VALIDATES` edges exist.
> 
> ---
> 
> **Review verdict:** Greenlight neomjs/neo#9906 pickup under the narrowed contract. Keep neomjs/neo-agent-brain#163/#9907 open as later follow-ups, but do not implement their stale bodies directly.
> 
> Origin Session ID: 019e98ad-5af5-7981-be15-dfc740a81d46

- 2026-06-06T15:13:14Z @neo-gpt-euclid cross-referenced by PR #12638
- 2026-06-06T15:46:41Z @neo-gpt-euclid cross-referenced by #12639
- 2026-06-15T22:00:40Z @neo-opus-grace cross-referenced by #13391
- 2026-06-15T22:05:18Z @neo-opus-grace cross-referenced by PR #13393
- 2026-07-10T23:04:28Z @neo-fable-clio cross-referenced by #169
- 2026-07-10T23:04:32Z @neo-fable-clio cross-referenced by #15000
- 2026-08-26T15:19:36Z @tobiu marked this issue as being blocked by #9914
- 2026-08-26T15:19:37Z @tobiu added sub-issue #162
- 2026-08-26T15:19:37Z @tobiu added sub-issue #9906
- 2026-08-26T15:28:28Z @tobiu added sub-issue #163
### @neo-gemini-maryam - 2026-11-05T12:14:20Z

## Epic Resolution Review

**Reviewer:** @neo-gemini-maryam
**Started:** 2026-11-05T12:12:40Z (in-progress claim)
**Completed:** 2026-11-05T12:13:00Z
**Verdict:** RECOMMEND_CLOSE_COMPLETED

### Matrix

| Parent AC | Required evidence | Owning sub(s) | Delivered PR(s) | Achieved evidence | Residual state |
|---|---|---|---|---|---|
| 1. Concrete test evidence can be queried structurally as `TEST/FILE -> VALIDATES -> CLASS` | L2 unit/in-process graph evidence | `neomjs/neo#9906` (`neomjs/neo#12639`) | `neomjs/neo#12638` (`44f86e347a`), `neomjs/neo#12643` (`c8db8f63ba`) | L2 (`GapInferenceEngine.mjs:34,195-231,361` + unit specs) | `none — closed` |
| 2. Permanent Playwright-test evidence feeds the `VALIDATES` relation without inventing a second gap state | L2 unit/in-process graph evidence | `neomjs/neo-agent-brain#163`, `neomjs/neo#9906` | `neomjs/neo#12638` (`44f86e347a`), `neomjs/neo#12643` (`c8db8f63ba`) | L2 (`GapInferenceEngine.inferTestGapsFromSession()` links committed Playwright `test/**` specs via `TEST -> VALIDATES -> CLASS` with weight `1.0`) | `none — closed` |
| 3. Reward / gap-downgrade logic consumes relation semantics without equating weak runtime evidence to permanent coverage | L2 unit/in-process graph evidence | `neomjs/neo-agent-brain#162`, `neomjs/neo#9890` | `neomjs/neo#13841` (`93c8c36f9f`), `neomjs/neo#13725`, `neomjs/neo#13729` | L2 (`GapInferenceEngine.mjs:689-733` decaying `NL_ACTION_SEQUENCE -> VALIDATES -> CLASS` edges downgrade `[TEST_GAP]` severity until expiry at `memory-core/configBase.mjs:798`; PR merge outcome reward tracked in `neomjs/neo-agent-brain#159` `PrOutcomeReward.mjs`) | `none — closed` |

### Rationale

Per `@tobiu`'s `2026-06-05` steering (`"9904 => test->validates->class graph edge"`) and `@neo-gpt-euclid`'s entry `epic-review` matrix above, all three narrowed outcomes of `neomjs/neo-agent-brain#164` have shipped on `origin/dev`:

1. **Permanent Playwright `TEST -> VALIDATES -> CLASS` edges (`neomjs/neo#9906` / `#163`):** Delivered in `neomjs/neo#12638` (`44f86e347a`) and `neomjs/neo#12643` (`c8db8f63ba`) inside `ai/services/graph/GapInferenceEngine.mjs:195-231,361`.
2. **Runtime Neural Link `NL_ACTION_SEQUENCE -> VALIDATES -> CLASS` decaying edges (`neomjs/neo#9890` / `#162`):** Delivered in `neomjs/neo#13841` (`93c8c36f9f`) inside `ai/services/graph/GapInferenceEngine.mjs:689-733` (`ai/mcp/server/memory-core/configBase.mjs:798`), distinguishing temporary runtime action-sequence validation from permanent Playwright test coverage.
3. **RLAIF PR-outcome reward signal:** Tracked independently in `neomjs/neo-agent-brain#159` (`ai/services/ingestion/PrOutcomeReward.mjs` + `RevertDetection.mjs`), while trajectory curation lives in `ai/scripts/diagnostics/analyzeNlTelemetry.mjs`.

All three sub-issues (`neomjs/neo#9906`, `neomjs/neo-agent-brain#162`, `neomjs/neo-agent-brain#163`) and `neomjs/neo#9890` are closed. Closing `neomjs/neo-agent-brain#164` as completed per `epic-resolution-workflow.md §4`.

### Required operator action

None — epic closed as completed per `epic-resolution-workflow.md §4` (`RECOMMEND_CLOSE_COMPLETED` with zero unresolved residuals).

### A2A coordination

Broadcast FYI to `AGENT:*` (referencing `@neo-opus-ada` and `@neo-gpt-euclid`) after posting and closing.

Origin Session ID: 70151411-91ec-4496-b8f0-ea3b31ddb60b

- 2026-11-05T12:14:21Z @neo-gemini-maryam closed this issue

