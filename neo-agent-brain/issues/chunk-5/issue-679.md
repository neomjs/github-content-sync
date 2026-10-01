---
id: 679
title: 'First-run recipe: live step evaluation and one host-owned record'
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees: []
createdAt: '2026-10-01T13:04:30Z'
updatedAt: '2026-10-01T18:31:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/679'
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
---
# First-run recipe: live step evaluation and one host-owned record

## Context

Epic neomjs/neo-agent-institution#351 (graduated from neomjs/neo#18965 on 2026-10-01), solution points 1 and 2: one shared setup recipe, evaluated live, and two renderers over one host-effect module. This leaf is the recipe, the record and the first renderer (the CLI); the cockpit renderer is the Institution leaf that projects what this one owns. ADR 0041 (*the bootstrap record and the verified-plane handoff*, filed beside this leaf as its own Brain ticket) is its authority and its merge gate.

Parent: neomjs/neo-agent-institution#351 (sub-issue link set after creation).

The Brain already holds the two halves in separate places. `ai/scripts/setup/initServerConfigs.mjs` is the per-clone config bootstrap: first-time copy of each `config.template.mjs`, regex drift detection, fail-closed on per-server drift — it decides nothing about a plane. `ai/scripts/maintenance/materializeDeploymentPrescriptions.mjs` is the host-side effect ledger for recovery knobs: a run id, a schema version, a manifest snapshot taken before Docker runs, a receipt written only after the carrier's digest still matches, under `~/.neo-ai/deployment-prescriptions` (`NEO_HOST_DEPLOYMENT_PRESCRIPTION_ROOT`). That is the record discipline ADR 0041 asks for, already proven on one effect class; the first run has no ledger at all — an operator following `SharedDeployment.md` performs 74-name effects by hand and nothing records intent, consent or what was done.

## The Problem

An outside operator's first run must reach a served plane through steps whose status is true now, not remembered. Today every step lives in prose, every effect is manual, and a resumed run cannot tell an accepted effect from one that never ran — the exact failure ADR 0041's witness is written against (accept an effect, interrupt, resume through the other renderer, answer from a wrong plane).

## The Architectural Reality

- Config authority is `AiConfig` (ADR 0019): "is this value set" is a leaf read or its declared metadata, never `process.env` and never the record. The presets leaf (sibling, not yet filed) declares env values over the §10.7 profiles; this leaf consumes a preset's env set as opaque input.
- The plane's current readiness is already observed by owners: `ai/services/fleet/projectDeploymentStateForFleet.mjs` (`ok` · `stale` · `unavailable`, never a fabricated plane), the ingress healthcheck with its served `plane.id` (ADR 0019 §10.3), the deployment-state snapshot. A recipe step reads these; it never re-implements them.
- The host state root is `~/.neo-ai/` — the prescriptions root and the plane tooling's secrets already live there; a checkout is reachable by `git clean -x`, the plane's data root does not exist before the plane, browser storage is per viewer.
- The vessel's main process and the CLI are the two callers with host authority (compose, ports, files); the cockpit page has none (epic point 2; D#18965 option C rejected).

## The Fix

1. **The recipe module** — one exported step list, versioned (`recipeVersion`), each step `{id, kind: 'question' | 'effect' | 'observation', evaluate(target) → {status, reason, observedAt}}`. `evaluate` is a fresh read for the bound target; no step reads the record for its status. v1's steps, in order: placement (where the plane runs) · preset · plane credential (PAT) · [advanced, folded] · effects: env file + secret files + compose up · observation: the served plane identity matches the target · validation: one provider call and one observed embedding (dimension included) · done: a query answered and first persistence (neomjs/neo-agent-brain#86's bar, J3 in neomjs/neo#14781).
   **The placement step's headroom rule** *(added 2026-10-01 from the #685 / #686 build)*: the step reads `probePlacement()` (#685) and `fitsPreset(probe, preset.workload)` over `presets` (#686), both threshold-free by contract, and adds the one judgement they do not make — a named headroom. `fitsPreset` reports raw margins; the step recommends a preset only when `margins.host ≥ headroomBytes` (v1: 4 GiB, the working headroom the 2026-09-23 harness-stack measurement needs before the OS starts compressing) and the guest margin is non-negative, offers a preset that fits by less than that as *possible, not recommended* with its margin shown, and never offers a `candidate` preset (no recorded floor) by default. Measured consequence that fixed the rule: with real model sizes `local-small` fits a 32 GiB host (14 GiB other use, 16 GiB VM) by 0.3 GiB on arithmetic — the 32 GiB tier's steer toward `hosted` lives here, not in the table or the probe. A `pressure: 'swapping'` host gets no local recommendation at all (the probe already refuses the fit).
2. **The record** — one JSON file per run under the host state root (`~/.neo-ai/setup/<runId>.json`, root overridable the way the prescriptions root is), `{schemaVersion, runId, target: {planeId | null before create, endpoint}, recipeVersion, consents: [...], receipts: [{effectId, acceptedAt, digest, outcome}]}`; secret-free by construction (references only). One writer: the host-effect module. A resumed run replays nothing whose receipt is `accepted`; an ambiguous effect is `reconcile-required` until a fresh matching observation settles it.
3. **The host-effect module** — the effect handlers (write env, write secret files with mode 0600, compose up, probe a port), each naming an executable local handler or an explicit operator action; imported by the CLI here and by the vessel's main process in the Institution leaf. One implementation.
4. **The CLI renderer** — a script beside `ai/scripts/setup/initServerConfigs.mjs`: prints the evaluated steps, asks the three questions, runs effects through the module, re-evaluates after each; `--json` for the vessel's smoke and the witness. It can run before any Brain container exists.
5. **Handoff** — once the served identity matches the run's target, current readiness comes from the plane's owners and the record answers only prior consent and effects (ADR 0041 §2.5). A target or recipe-version change retires every receipt as current proof (§2.7).

Placement (structure map run at filing, `npm run ai:structure-map -- --files --loc`, exit 0): the CLI goes beside `ai/scripts/setup/initServerConfigs.mjs` (sibling fast-path). The recipe/record/effects module: `ai/services/fleet/` is the sibling-pattern candidate — it already holds the host-side provisioning pair (`provisionAgentRepo.mjs`, `startAgentProvisioned.mjs`) and the plane readers (`planeDeploymentStateReader.mjs`, `projectDeploymentStateForFleet.mjs`); `ai/services/` has no `setup/` folder today, so a new folder is a novel choice that runs the full structural pre-flight in the PR, cited there. The implementer decides at the PR with the citation; the design seat reviews it.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| the recipe module's `evaluate(target)` | ADR 0041 §2.3; epic point 1 | fresh status per step for the bound target and recipe version; unknown version → mismatch shown, nothing inferred | a failed observation is `unknown` with its reason, never green | module JSDoc | unit: every step's status is a function of injected observers, none of the record |
| the record file under the host state root | ADR 0041 §2.1–2.2 | one writer; secret-free; run-scoped; `accepted` receipts never replay | an unreadable record → the run starts fresh and says so | module JSDoc + a `SharedDeployment.md` pointer | unit: the §3 witness (accept · interrupt · resume · wrong plane) |
| the host-effect handlers | ADR 0041 §2.6 | each effect names a local handler or an operator action; remote effects wait for neomjs/neo-agent-brain#83 | an effect without a handler is an operator action, never skipped | handler JSDoc | unit per handler with a fake host; the compose handler behind a flag in CI |
| the CLI (`--json`) | this ticket | the three questions, effects through the module, re-evaluate after each | non-tty → `--json` only, no prompts | script `--help` | spec: a cold run against a fake host reaches the observation step |
| target binding | ADR 0041 §2.4; neomjs/neo-agent-institution#181 | create binds the declared `plane.id`; attach binds the served identity, endpoint is a coordinate | mismatch → fail closed (`assertServedPlane`) | JSDoc | unit: A's receipts turn no B step green |

## Acceptance Criteria

- [ ] AC-1 A step's status is a fresh evaluation: with the record holding `accepted` receipts and the observers stubbed to fail, every step reads `unknown` / `failed`; with the observers green and an empty record, the observation steps read `ok`. Unit.
- [ ] AC-2 The ADR 0041 §3 witness: accept an effect, interrupt before its receipt, resume — the effect is `reconcile-required`, not replayed and not green; a fresh matching observation settles it. Unit, red-first.
- [ ] AC-3 Target binding: receipts and consents bound to target A turn no step green for target B; a recipe-version change retires them as current proof and they stay readable as history. Unit.
- [ ] AC-4 The record holds no secret: a fake preset with a PAT and a provider key produces a record that contains neither string; the secret files are mode 0600. Unit.
- [ ] AC-5 The CLI on a fake host: three questions, `--json` output lists every step with status and reason, exit code reflects the terminal step. Spec.
- [ ] AC-6 *(post-merge, on the first outside host)* a cold run reaches the observation step against a real compose; the density count (decisions + manual actions) recorded on the epic.

## Out of Scope

The presets and the probe (sibling leaves); the `*File` credential adapter (sibling); the cockpit renderer and the vessel's IPC (Institution leaf); remote host effects (neomjs/neo-agent-brain#83); guides (neomjs/neo-agent-brain#86).

## Avoided Traps

A stored "completed" bit (the cockpit's `● streaming` over a three-week-old row). A machine writer of `config.mjs` (falsified on D#18965 — env values and secret files only). Deriving readiness from receipts. A second effect implementation in the vessel. A plane-owned record for steps that create the plane. Endpoint-keyed binding (neomjs/neo-agent-institution#181).

## Related

neomjs/neo-agent-institution#351 (parent) · ADR 0041 (gate; its Brain ticket filed in the same turn) · ADR 0019 §§10.3 / 10.7 · neomjs/neo-agent-brain#83 · neomjs/neo-agent-brain#86 · neomjs/neo#14781 (J3) · neomjs/neo-agent-institution#12 · neomjs/neo-agent-institution#181 · `ai/scripts/maintenance/materializeDeploymentPrescriptions.mjs` (the receipt discipline precedent) · `ai/scripts/setup/initServerConfigs.mjs` (the config bootstrap this wraps, never replaces)

Decision Record: Required: ADR 0041 (Accepted at D#18965's quorum; its PR must merge before this leaf's)
Decision Record impact: depends-on ADR 0019; aligned-with ADR 0026 (the actuator is not an install RPC)

unowned-rationale: a build lane that ranks behind the Claude Desktop seat path by the operator's order of 2026-10-01; offered by DM to @neo-opus-ada and @neo-opus-grace for after their current neomjs/neo-agent-brain#571 subs; the design seat (author) answers questions on this ticket and reviews the PR.

Sweeps: live latest-open sweep — the latest 20 open Brain issues read at 2026-10-01T13:02:42Z (newest #675), no equivalent; A2A in-flight sweep — the last 30 messages at 13:03Z, all read-states, no `[lane-claim]` / `[lane-intent]` on the first-run recipe or record (open claims are seat-side: #675, #674, #621); Memory Core rationale sweep (`query_raw_memories` on the problem's nouns) surfaced no decision beyond D#18965 itself (OQ1 `DC_kwDODSospM4BG0hx`, OQ5 `DC_kwDODSospM4BG0yd`, option H/I folds); own-assignment sweep — #37, #50, #51, #53, none on this surface; structure map cited under Placement.

Origin Session ID: 6682a116-897e-4c18-925e-4320d0489481
Retrieval Hint: "first-run recipe host record consent receipts evaluated live CLI renderer setup wizard"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 6682a116-897e-4c18-925e-4320d0489481


## Timeline

- 2026-10-01T13:04:31Z @neo-fable-clio added the `enhancement` label
- 2026-10-01T13:04:31Z @neo-fable-clio added the `ai` label
- 2026-10-01T13:04:31Z @neo-fable-clio added the `architecture` label
- 2026-10-01T13:04:31Z @neo-fable-clio added the `agent-os` label
- 2026-10-01T13:05:23Z @neo-fable-clio added parent issue #351
- 2026-10-01T13:08:49Z @neo-fable-clio cross-referenced by PR #680
- 2026-10-01T13:31:25Z @neo-fable-clio cross-referenced by #685
- 2026-10-01T13:32:10Z @neo-fable-clio cross-referenced by #686
- 2026-10-01T13:38:29Z @neo-fable-clio cross-referenced by #384
- 2026-10-01T15:15:27Z @neo-fable-clio cross-referenced by #696
- 2026-10-01T15:16:05Z @neo-fable-clio cross-referenced by #697
- 2026-10-01T16:41:30Z @neo-opus-vega cross-referenced by PR #705
- 2026-10-01T18:21:27Z @neo-fable-clio cross-referenced by #713
- 2026-10-01T18:21:51Z @neo-fable-clio cross-referenced by #714
- 2026-10-01T18:23:30Z @neo-fable-clio cross-referenced by PR #715
- 2026-10-01T18:30:26Z @neo-fable-clio cross-referenced by #351
- 2026-10-01T18:51:27Z @neo-gpt cross-referenced by PR #707

