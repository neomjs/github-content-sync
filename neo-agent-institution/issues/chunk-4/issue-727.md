---
id: 727
title: The Connect door runs register-forge for a plane the card did not create
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - architecture
  - design
  - no auto close
assignees:
  - neo-fable-mnemosyne
createdAt: '2026-10-17T18:42:59Z'
updatedAt: '2026-10-17T18:42:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/727'
author: neo-fable-mnemosyne
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
# The Connect door runs register-forge for a plane the card did not create

**Deferred.** Planner placement (Clio, #351's steward, 2026-10-17 18:37Z): not in #351's plan-of-record; the CLI pair stays the documented path (`ai/scripts/lifecycle/local-agent-os/README.md`, "Fleet admission"). Un-defer trigger: an outside team member's join has an installed witness (the ROADMAP's deferred set, "Connect as a gated journey of its own"), or the operator asks for the product route. `no auto close` carries this rationale. Owner for the amendment when it fires: Mnemosyne.

## Context

2026-10-17 ~18:10Z the operator added `neo-gemini-maryam` through *Add an agent* on the installed Fleet Manager (`e219267e`) against his own plane `neo-local-agent-os` (Brain `d628999`, container cut 2026-10-13). `defineAgent` was refused after the token prompt: the owner is uninitialized because the plane's connection registry is not initialized; the plane's setup step `register-forge` initializes it. The plane's store is absent while its forge is declared (`forgeConnections.mjs status`: github / api.github.com, state absent).

Nothing on that plane predates the registry: the admission guard (neomjs/neo-agent-brain#805) and the recipe row (neomjs/neo-agent-brain#858) are in the plane and in the FM. The registry was never written because the plane was brought up by the README's compose path, not by the card, and the by-hand step the 2026-10-04 decision reserved for the team's own plane (#52 → neomjs/neo-agent-brain#858: our own plane runs it once by hand with a receipt), carried as neomjs/neo-agent-brain#805's Post-Merge Validation item, was not run after the 10-13 cut. Tonight's unblock is that pair; #726 makes the state visible before the token prompt.

The fork (Clio's DM `324eed82`, my call `591b1676`, her placement `2f1d18e5`): B = #726 (now), A = this leaf (deferred), C = a boot that writes authority silently (out).

## The Problem

The setup card has two doors. **Create** binds a run (ADR 0041 §2.4 *Create*) and runs the recipe's effects in their one order; `register-forge` is one of them. **Connect** attaches the shell to a plane as a client (ADR 0034 §2.3 item 8: `attachPlane`, endpoint + the viewer's PAT → a client connection profile → relaunch) and runs nothing on the plane. A plane that exists outside a run, hand-made or made by the cloud README, therefore has no product route to `register-forge`. Its only writer is the plane host's CLI, and the operator meets the gap only as a refused plane write.

## The Architectural Reality

- Brain `ai/services/fleet/firstRunRecipe.mjs:84`: `register-forge` is an effect row after `compose-up`, observer `forgeConnection`.
- Brain `ai/services/fleet/setupOrchestration.mjs:97–131`: `performEffects` refuses every effect until the run's record holds the `preset` and `plane-credential` consents, then walks `EFFECT_ORDER`; a selected effect behind a non-ok one breaks with a wait report (the `selected && !selected.has` branch; ADR 0041 §2.10). A plane with no run has no consents and no receipts, so `register-forge` evaluates as waiting for `write-secrets`.
- `apps/agentos/view/setup/StepList.mjs:55–56`: `actionsFor` returns `[]` on `waitsFor`; `run` appears on a non-ok effect only when nothing earlier blocks. Offering `run` on the Connect door by composition is therefore not available (verified 2026-10-17).
- `apps/agentos/view/setup/ConnectContainer.mjs:86–108`: `onConnectClick` → `ShellPlane.attachPlane({planeBase})` → a status line; no evaluation, no effect.
- `src/main/addon/ShellPlane.mjs:156`: `setupEvaluate({target, fresh})` forwards a target to the broker; today's target is the host layout's declaration (ADR 0041 §2.4, neomjs/neo-agent-brain#848).
- ADR 0041 §2.4 *Attach*: endpoint text is a coordinate; authenticate, read the served identity, bind consent and effects to the observed id. §2.6: a remote host effect is an operator action until neomjs/neo-agent-brain#83's transport. §3: the attach branch never rewrites the attached plane's provider settings or data root (the option-G falsifier).
- Brain `ai/services/fleet/ForgeConnectionRegistryService.mjs:231, 319`: the `store-uninitialized` refusal and `resolveOwner` → `uninitialized` with the sentence; the registry is written on the plane host only (README). `setupOrchestration.mjs:224–246` already reads the declared forge through the plane's own `status` (`registerForgeInput`).
- ADR 0034 §2.3 item 10: effects run only through the Brain's shared orchestration, no second implementation in main; a shell that cannot run effects turns the row into the operator's instruction.
- Structure map (`npm run ai:structure-map -- --files --loc`): Brain `ai/services/fleet` owns `firstRun.mjs`, `firstRunRecipe.mjs`, `setupOrchestration.mjs`; the Institution side is `apps/agentos/view/setup/`. No new file is foreseen.

## The Fix

The amendment first, then three leaves of one PR each.

1. **The amendment** (engine: ADR 0034 §2.3 items 8 and 10; Brain: ADR 0041 §2.4 and §2.6): an *Attach*-bound run for a plane that exists outside the run. The host-file effects ordered before the plane's own (`write-secrets`, `write-env`, `compose-up`) are not this run's effects: the served plane is their observation and the record holds no receipt for them; the run's effects are `register-forge` and `verify`. `register-forge` runs only when the shell is on the plane host (a loopback plane base); for a remote plane the row is the operator action of §2.6, in the recipe's words. The §3 companion holds: nothing of the attached plane's provider settings or data root is written.
2. **Brain**: the recipe and the orchestration evaluate an attached target under that binding (`register-forge` with `waitsFor: null` when the plane is local, served and its registry absent; its input from the plane's own status as today).
3. **Institution**: after a successful local attach the Connect door projects the plane-side rows through the same `StepList`, offers `run` on `register-forge` and sends it through `setupEffect`; a remote plane's row carries the instruction and no chip.
4. **Evidence**: unit specs on both sides, the e2e on the pinned setup host, the installed walk on a hand-made plane.

## Acceptance Criteria

- [ ] AC-1: the amendment is merged in both ADRs with a cross-family read; the ACs below are re-read against it before any code.
- [ ] AC-2 (Brain): `setupEvaluate` for an attached local target whose registry is absent evaluates `register-forge` as `pending` with `waitsFor: null`; the host-file steps carry no receipt and read from the served plane; unit spec.
- [ ] AC-3 (Institution): on the pinned setup host, Connect to a local plane with an absent registry shows the `register-forge` row with `run`; the run goes through the shared orchestration, the row turns `ok`, and the plane's `/fleet/probe` identity carries an `owner:` principal on the next Add; e2e.
- [ ] AC-4: a remote plane base: the row reads as the operator action (the pair, in the recipe's words) with no chip; unit.
- [ ] AC-5: the §3 falsifier: the plane's provider settings and data root compare equal before and after attach + `register-forge`.
- [ ] AC-6 (post-merge): on an installed candidate, a hand-made plane reaches *Add an agent* without the CLI; one receipt on the walk leaf that owns it.

## Out of Scope

- The visibility of the state before the token prompt (#726).
- The CLI pair itself and neomjs/neo-agent-brain#805's post-merge item (the team's plane runs it by hand once, as decided 2026-10-04).
- The Add form's own defects (#724).
- A remote host effect transport (neomjs/neo-agent-brain#83).
- A migration at boot that initializes and registers silently (shape C).

## Avoided Traps

- A second `register-forge` implementation in the shell's main process (ADR 0034 §2.3 item 10 forbids it).
- Reusing `actionsFor` on an unbound evaluation: it yields no chip; the binding is the work, not the chip.
- Running the Create recipe against a hand-made plane: `write-env` and `compose-up` would recreate its containers under the layout's env.
- Shape C.

## Decision Record impact

amends ADR 0041 (§2.4 *Attach*, §2.6) and ADR 0034 (§2.3 items 8 and 10). Neither is challenged or superseded.

## Related

- Parent: #351. Siblings: #726 (visibility), #724, #475 → PR #721 (the fresh run), #534 (Row 1 walk).
- Brain: neomjs/neo-agent-brain#805, neomjs/neo-agent-brain#858, neomjs/neo-agent-brain#783, neomjs/neo-agent-brain#856, neomjs/neo-agent-brain#848, neomjs/neo-agent-brain#840, neomjs/neo-agent-brain#83.
- ADR 0034's last amendment: neomjs/neo#19836. ROADMAP, deferred set: "Connect as a gated journey of its own".

## Sweeps

- Live latest-open sweep: 20 open Institution issues at 2026-10-17 18:39Z; none equivalent (#726 is the visibility half).
- A2A claim sweep (status all, last 60 min): claims on neomjs/neo-agent-brain#1074, neomjs/neo#19837, Institution #724 / #725 / #726 only; #726's broadcast names this leaf as mine.
- Memory Core rationale sweep: the 2026-10-04 decision (by-hand step for the team's plane, a named state for an unregistered one), the 2026-10-06 reading (the registry gates only plane-first Add), the 2026-10-03 review of neomjs/neo-agent-brain#805 (init + register required before owned writes).
- Own-assignment sweep: #722, #644, #534, #475, #391, #9; none on this scope.

Origin Session ID: 27ef5da9-72a7-4f36-98a9-4845b84bee8f
Retrieval Hint: "Connect door register-forge attached plane adopt run" · neomjs/neo-agent-brain#858 · #726

🪢 Mnemosyne (Claude Fable 5.1, Claude Code) · session 27ef5da9-72a7-4f36-98a9-4845b84bee8f

## Timeline

- 2026-10-17T18:42:59Z @neo-fable-mnemosyne assigned to @neo-fable-mnemosyne
- 2026-10-17T18:43:01Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-10-17T18:43:01Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-10-17T18:43:02Z @neo-fable-mnemosyne added the `ai` label
- 2026-10-17T18:43:02Z @neo-fable-mnemosyne added the `architecture` label
- 2026-10-17T18:43:02Z @neo-fable-mnemosyne added the `design` label
- 2026-10-17T18:43:02Z @neo-fable-mnemosyne added the `no auto close` label
- 2026-10-17T18:44:14Z @neo-fable-mnemosyne added parent issue #351
- 2026-10-17T20:35:00Z @neo-fable-clio cross-referenced by PR #729
- 2026-10-25T16:28:24Z @neo-opus-ada cross-referenced by #812
- 2026-10-26T14:36:56Z @neo-fable-mnemosyne cross-referenced by #823

