---
id: 686
title: 'Three supported presets as env sets: hosted, local-small, local-full'
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-fable-clio
createdAt: '2026-10-01T13:32:09Z'
updatedAt: '2026-10-01T21:10:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/686'
author: neo-fable-clio
commentsCount: 1
parentIssue: 351
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 713 The Gemini model leaves gain env bindings so the hosted preset can name its models'
blocking: []
---
# Three supported presets as env sets: hosted, local-small, local-full

## Context

Epic neomjs/neo-agent-institution#351, shape point 4 (*simple by default: choose a supported preset, supply what it genuinely lacks*) and the Discussion's option I (*a curated preset recipe over declared leaves*, adopted for the first cut). Three Discussion questions were dispositioned onto this leaf at graduation: **OQ4** (the embedding model is a birth decision — the pinned dimension per preset and the re-embed path, declared before any preset ships), **OQ8** (a quality floor per preset, measured before a preset is called supported), and the threshold half of **OQ3** (thresholds come from measured presets, never from our plane). OQ5's answer binds the shape: presets are **sets of ENV values over ADR 0019 §10.7's declared profiles**, each declaring `authorityProfile`; never leaf defaults (§10.8), never a machine-written `config.mjs` (falsified on D#18965).

Parent: neomjs/neo-agent-institution#351 (sub-issue link set after creation). Consumers: the recipe's preset step (#679) and the probe's `fitsPreset()` (#685).

*Narrowed 2026-10-01 by the author: the AiConfig half (AC-4, the Gemini env bindings) moved to #713 and the floor instrument with its recorded runs (AC-5 / AC-6) to #714, so the `ai/configBase.mjs` touch gets its own ADR 0019 review and the measurement lane its own receipts. This leaf is the table, the parity lint and the re-embed path.*

## The Problem

Today the supported configurations live in prose across `learn/agentos/ModelProviders.md`, `SharedDeployment.md` and `DeploymentCookbook.md` — 74 distinct `NEO_*` names (D#18965 Evidence) — and the only measured footprint is our own plane (≈ 31 GB with local inference and a 4096-dim corpus). An outside operator cannot tell which three or four values matter, which model pair is known to work on the shipped Tri-Vector path, or that the embedding dimension is pinned per collection and a later preset change means a new collection. Nothing mechanical notices when the prose drifts from the leaves it describes.

## The Architectural Reality

- Config authority is `AiConfig` (ADR 0019): a preset may only set env values for declared leaves' bindings (§10.8); `config-leaf-parity.json` is a profile-specific lint precedent, not the registry; the Gemini `modelName` / `embeddingModel` Tier-1 defaults are the two first-run values WITHOUT an env binding (#713).
- Measured on a fresh small institution (2026-09-23, `fm-fresh-small` beside the live plane, 213-file tenant list + the 47k-file corpus slice): the plane idles at 0.39 GiB and peaks ≤ 2.5 GiB; the 0.6b embedder (1024 dims) finishes the 213-file backlog in ≈ 86 s and embeds 495 corpus chunks per 5-min slice; the 8b embedder needs ≈ 414 s and 110 per slice while returning the better documents; disk ≈ 150 MB either way at that scale.
- Chat-model floor on the shipped Tri-Vector path (2026-09-23): gemma-4-26b-a4b runs it and sets the floor (4–5 grounded nodes, 0 dangling edges per session document); gpt-oss-20b is 3.7× faster on prefill but thin below that floor; Qwen3.6 is blocked by LM Studio's reasoning-channel handling. The instrument that reproduces this is #714.
- `NEO_VECTOR_DIMENSION` is pinned per collection (default 4096; 3072 for `gemini-embedding-001`; 1024 for the 0.6b) and a mismatch fails (`SharedDeployment.md`).
- Providers: `NEO_MODEL_PROVIDER` / `NEO_EMBEDDING_PROVIDER` ∈ `openAiCompatible` · `gemini` · `ollama`; the hosted path ran this project before its local models.
- Harness logins are never part of a preset (operator steer, folded); the plane's own credentials (PAT; a provider key only in the hosted preset) are the credential-step leaf's business, not this one's.

## The Fix

1. **A preset table as data** — one module exporting `presets` beside the probe in `ai/services/fleet/`: `{id: 'hosted' | 'local-small' | 'local-full', label, inference, profile (ADR 0019 §10.7), authorityProfile, env: {NEO_…: value}, requires: ['chatModel' | 'embeddingModel' | 'providerKey' | 'repos' | 'pat'], vectorDimension, embedder, chatModel, workload: {planeIdleBytes, planePeakBytes, modelsBytes, vmCapRecommendedBytes}, qualityFloor: {instrument, measuredAt, result} | null}`. Values measured on the fixture plane, never on ours; `workload` is what the probe's `fitsPreset()` consumes. The hosted preset declares `pendingBindings` for the two Gemini leaves until #713 lands.
2. **The leaf-parity lint** — a spec that resolves every `env` key of every preset against the declared leaf bindings (`ai/configBase.mjs`'s `leaf(…, 'ENV_NAME')` declarations) and fails on an unknown key: the mitigation named in the Discussion's convergence pass for option I ("drift between the recipe and the leaves").
3. **The birth decision** — each preset declares `vectorDimension`; the recipe's preset step (#679) shows it; a preset change after ingest is a **new store**, never an in-place re-dimension — the module exports `reembedPath(fromPreset, toPreset)` returning the named path (new store name, the env values that select it, the re-embed steps) or `null` when the dimension is unchanged.
4. **The Gemini bindings** — #713.
5. **The floor instrument** — #714.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `presets[]` | D#18965 option I + OQ3/4/8 dispositions; ADR 0019 §10.7/§10.8 | env sets over declared profiles, each with `authorityProfile`, workload, dimension, floor | a preset without a recorded floor is `candidate`, never offered by default | module JSDoc + `ModelProviders.md` pointer | unit: three presets, every field present |
| the leaf-parity lint | convergence pass (option I's residual risk) | every preset env key resolves to a declared leaf binding | an unknown key fails the suite | JSDoc | unit: a fixture preset with a typo'd key fails |
| `vectorDimension` + `reembedPath()` | OQ4 | pinned per preset; change after ingest → a new store by name | same dimension → `null` | JSDoc + `SharedDeployment.md` | unit: 1024 → 4096 names the path; 4096 → 4096 is `null` |
| the Gemini env bindings | OQ5's AC | → #713 | — | — | — |
| the floor instrument | OQ8 | → #714 | — | — | — |

## Acceptance Criteria

- [ ] AC-1 Three presets exported with every contract field; `local-small` pins 1024 dims with the 0.6b embedder, `local-full` the 8b at 4096, `hosted` 3072 with `gemini-embedding-001`; workloads carry the fixture measurements, not our plane's. Unit.
- [ ] AC-2 The leaf-parity lint resolves every preset env key against the declared leaf bindings and fails on a fixture preset with an unknown key. Unit.
- [ ] AC-3 `reembedPath()` names the new-store path for a dimension change and returns `null` for none; no code path re-dimensions in place. Unit.
- [ ] AC-4 *(after #713 / PR #716 lands — #686 is blocked-by #713; moved here from #713's AC-3 on 2026-10-01 18:4xZ per @neo-opus-grace's closure read)*: the hosted preset names both Gemini models through its `env` (`NEO_GEMINI_MODEL`, `NEO_GEMINI_EMBEDDING_MODEL`) and its `pendingBindings` is empty; the leaf-parity spec stays green. Unit, carried by PR #715 after its rebase.
- [ ] ~~AC-5 / AC-6~~ → #714 (the floor instrument; the recorded runs, hosted post-merge with an operator-supplied key).

## Out of Scope

The probe (#685); the credential step and the `*File` adapter (OQ5's custody half, #696); the cockpit's preset cards (Institution leaf); any new provider; the corpus ingestion itself; a model zoo (the Discussion's position: supported presets only); the Gemini bindings (#713) and the floor instrument (#714).

## Avoided Traps

Leaf defaults as presets (§10.8 forbids). A machine-written `config.mjs` (falsified). An in-place re-dimension. Deriving the dimension from the chosen model without an observed embedding (withdrawn on D#18965 — the recipe's validation step observes it). Our plane as the threshold source. Prose as the registry (the drift the lint exists for).

## Related

neomjs/neo-agent-institution#351 (parent) · #679 (the preset step consumes this table) · #685 / PR #707 (consumes `workload`) · #713 · #714 · neomjs/neo#18965 OQ3/OQ4/OQ5/OQ8 + option I · ADR 0019 §§10.7/10.8 · `learn/agentos/ModelProviders.md` · `learn/agentos/SharedDeployment.md` · neomjs/neo-agent-brain#86 (the guides consume the preset names)

Decision Record impact: depends-on ADR 0019 (critical gate: read before any `ai/` config touch — now #713's); aligned-with ADR 0041 (a preset is an input, never a stored status).

unowned-rationale: a build lane behind the Claude Desktop seat path (operator priority 2026-10-01); the ADR 0019 author (@neo-opus-grace) is the natural first refusal for the binding half (#713), the measurement half (#714) is the author's (the 2026-09-23 fixture receipts). *(Taken by @neo-fable-clio at 18:16Z for AC-1…AC-3.)*

Sweeps: live latest-open sweep — the latest 20 open Brain issues read at 2026-10-01T13:29:31Z (newest #684), no equivalent; A2A in-flight sweep — the last 12 messages at 13:29Z, all read-states, no `[lane-claim]` / `[lane-intent]` on presets or the embedding dimension; Memory Core rationale sweep (`query_raw_memories`) — the 2026-09-23 fixture-plane measurements and tier derivation (folded into D#18965's Graduation Criteria), nothing newer; own-assignment sweep — #678, #37, #50, #51, #53, none on this surface; structure map (`npm run ai:structure-map -- --files --loc`, exit 0, 13:0xZ) — cited under The Fix.

Origin Session ID: 6682a116-897e-4c18-925e-4320d0489481
Retrieval Hint: "three presets env sets declared leaves vector dimension birth decision re-embed new collection quality floor instrument"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 6682a116-897e-4c18-925e-4320d0489481



## Timeline

- 2026-10-01T13:32:11Z @neo-fable-clio added the `enhancement` label
- 2026-10-01T13:32:11Z @neo-fable-clio added the `ai` label
- 2026-10-01T13:32:12Z @neo-fable-clio added the `architecture` label
- 2026-10-01T13:32:12Z @neo-fable-clio added the `agent-os` label
- 2026-10-01T13:32:32Z @neo-fable-clio added parent issue #351
- 2026-10-01T13:38:29Z @neo-fable-clio cross-referenced by #384
- 2026-10-01T15:15:27Z @neo-fable-clio cross-referenced by #696
- 2026-10-01T15:16:05Z @neo-fable-clio cross-referenced by #697
- 2026-10-01T17:20:07Z @neo-fable-clio cross-referenced by PR #707
- 2026-10-01T18:16:12Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-01T18:21:27Z @neo-fable-clio cross-referenced by #713
- 2026-10-01T18:21:51Z @neo-fable-clio cross-referenced by #714
- 2026-10-01T18:23:30Z @neo-fable-clio cross-referenced by PR #715
- 2026-10-01T18:30:26Z @neo-fable-clio cross-referenced by #351
- 2026-10-01T18:31:05Z @neo-fable-clio cross-referenced by #679
- 2026-10-01T18:33:23Z @neo-opus-grace cross-referenced by PR #716
- 2026-10-01T18:45:26Z @neo-fable-clio marked this issue as being blocked by #713
- 2026-10-01T19:06:46Z @neo-fable-clio cross-referenced by #685
- 2026-10-01T19:08:55Z @neo-fable-clio referenced in commit `bc01ae7` - "feat(fleet): three supported presets as env sets over declared leaves, with the re-embed path (#686)"
- 2026-10-01T19:23:28Z @neo-fable-clio referenced in commit `78c7efc` - "feat(fleet): three supported presets as env sets over declared leaves, with the re-embed path (#686)"
- 2026-10-01T19:23:28Z @neo-fable-clio referenced in commit `6400d7c` - "feat(fleet): the hosted preset names its Gemini models through the new env bindings (#686)"
- 2026-10-01T20:06:13Z @neo-fable-clio referenced in commit `db7d749` - "feat(fleet): three supported presets as env sets over declared leaves, with the re-embed path (#686)"
- 2026-10-01T20:06:13Z @neo-fable-clio referenced in commit `d4b58c6` - "feat(fleet): the hosted preset names its Gemini models through the new env bindings (#686)"
- 2026-10-01T20:36:10Z @neo-fable-clio cross-referenced by PR #732
- 2026-10-01T20:55:00Z @neo-fable-clio referenced in commit `b7dd6d8` - "fix(fleet): the presets speak the profile's consumed inputs and a dimension change needs a validated fresh database (#686)

Review repair. The local overlay's provider anchor fixed all three providers and read the openAiCompatible model names from its own NEO_LOCAL_AGENT_OS_* inputs, and neither Compose file forwarded NEO_VECTOR_DIMENSION or NEO_CHROMA_DATABASE, so a preset's provider and dimension choices never reached a container. The anchor now takes the providers as inputs with the local default and forwards the dimension and the database; the presets name the inputs the profile reads; the parity witness scans the effective profile (last writer per env name across the files) beside the leaf names. reembedPath no longer derives a store from the dimension: it requires an explicit fresh database name, refuses the current one and the default, emits the two env values the profile forwards, and leaves existence in Chroma to the deployment's own check."
- 2026-10-01T21:08:51Z @neo-fable-clio cross-referenced by PR #736
### @neo-fable-clio - 2026-10-01T21:10:58Z

## Session handover (sunset 2026-10-01 ~21:10Z) — owner @neo-fable-clio

**State:** PR #715 (Resolves #686) at `b7dd6d8`, 28/28 checks green, Emmy's REQUEST_CHANGES (5385288341) answered in one commit — the local overlay's provider anchor takes the three providers as inputs and forwards `NEO_VECTOR_DIMENSION` + `NEO_CHROMA_DATABASE`; the presets speak the profile's consumed inputs; the parity witness scans the effective profile (last writer per env name); `reembedPath` requires a validated fresh database name. Author response: comment 5940363165; re-review requested from @neo-gpt-emmy (DM 21:04Z).

**Pickup protocol:** `gh pr view 715` → if approved, post the merge-handoff broadcast (human-only merge); if a second round lands, answer it (1× RC then converge — a third round is a signal to converge in a DM, not another commit cycle). Two PRs stack on this one (#732, #736): after the merge, rebase both onto dev.

**Empirical anchors:** 24/24 local (presets 5 + probe 21 − overlap), `lint-config-template-ssot` green with the compose census unchanged, `check-aiconfig-antipatterns` 0 new, archaeology 0. Emmy's method to keep: resolve the YAML merge anchors and execute the pure helper — a declared binding proves a name exists, not that a profile forwards it.

**Known limit to offer if asked:** the profile parity is a text scan (no YAML library) modelling the anchor merge as last-writer-per-name; a YAML-resolved sibling arm is a 30-line addition.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 6682a116-897e-4c18-925e-4320d0489481



