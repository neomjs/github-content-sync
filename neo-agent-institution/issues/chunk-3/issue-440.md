---
id: 440
title: 'The setup card''s run and re-check actions reach the vessel''s effect channel, and the first completed run records its density'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-fable
createdAt: '2026-10-02T13:04:28Z'
updatedAt: '2026-10-02T18:29:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/440'
author: neo-fable
commentsCount: 0
parentIssue: 351
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 451 Pin the merged Brain setup and Fleet contracts'
  - '[x] 750 The first-run recipe''s effect orchestration leaves the CLI so the vessel''s setup broker runs the same effects'
blocking: []
---
# The setup card's run and re-check actions reach the vessel's effect channel, and the first completed run records its density

## Context

The follow-up of #384 (the setup card's Create door, in PR) under Epic #351. The card's `run` and `re-check` actions call `shell-setup-effect`; the vessel's broker (`harness/setupBroker.mjs`) refuses that one channel by name (`EFFECT_UNWIRED_REASON`) because the recipe's effect orchestration (`performEffects`, `settlePending`) is module-private to the Brain CLI's `main()`. neomjs/neo-agent-brain#750 moves both into `ai/services/fleet/setupOrchestration.mjs`, exported with an `effectIds` filter; this leaf consumes them once an Institution Brain pin carries #750. Until then the card hands the operator the CLI's command and counts it as a manual action — honest, and the epic's rule (one host-effect module, never two) holds.

## The Problem

On the installed Fleet Manager the three effect rows (`write-env`, `write-secrets`, `compose-up`) cannot be consented to from the card, and an interrupted effect's `re-check` only re-evaluates (the settle pass is the CLI's). The epic's density target (point 7: three decisions + three consents, zero manual actions on a Docker host) cannot be met from the cockpit renderer, so #351's AC-6 receipt — the first completed run's count — has no producer.

## The Architectural Reality

- `harness/setupBroker.mjs#effect` admits the sender, loads the modules and refuses; `#evaluate` already composes the CLI's layout and observers; `#answer`/`#credential` already record consents through `hostEffects.recordConsent`.
- `apps/agentos/view/setup/CreateContainer.mjs#runEffect` handles both replies: a refusal becomes the status line's instruction (`manualActions++`); an evaluation replaces the rows. `onStepClick`'s `re-check` is a plain re-evaluation today.
- `countDensity` (same file) counts answered questions and accepted receipts; `firstPersistence` fires once with the count; the Viewport controller hides the card on it.
- ADR 0041 §2.3/§2.6: an effect runs once after consent; an interrupted effect is settled by a matching observation, never replayed.

## The Fix

1. `setupBroker#effect`: `performEffects({…, effectIds: [effectId], report})` over the resolved run, then the re-evaluation; a reported refusal is the reply's `reason`.
2. `setupBroker#evaluate` runs `settlePending` before evaluating when the previous evaluation held a `reconcile-required` row, so `re-check` settles through the CLI's rule.
3. `SETUP_MODULE_PATHS` gains `orchestration: 'ai/services/fleet/setupOrchestration.mjs'`; `EFFECT_UNWIRED_REASON` retires with its arm.
4. The density receipt: the Viewport controller's `onSetupFirstPersistence` posts nothing itself — the e2e on the fixture plane reads `setupRun.decisions` / `manualActions` from the provider and the run's count is recorded on #351 by the lane that first completes a run (the epic's AC, a human-readable comment).

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `shell-setup-effect {effectId}` (existing channel) | neomjs/neo-agent-brain#750's `performEffects` | Runs the one consented effect in the CLI's order rules; replies the re-evaluated run. | A refusal (preset env, credentials) → `{ok: false, reason}` in the orchestration's words; `reconcile-required` ahead of it → refused, never replayed. | JSDoc | `setupBroker.spec.mjs`: one-effect run · refusal · the halt |
| `shell-setup-evaluate` (existing) | #750's `settlePending` | Settles interrupted effects by a matching observation before evaluating. | Nothing to settle → unchanged. | JSDoc | `setupBroker.spec.mjs` resume arm |
| `CreateContainer#runEffect` (existing) | #384 | Unchanged contract; the manual-action path stays for a refusal. | — | — | `createContainer.spec.mjs` (existing arms stay green) |

## Acceptance Criteria

- [ ] AC-1 With a Brain pin carrying neomjs/neo-agent-brain#750, `run` on `write-env` writes the carrier through the host-effect module and the row re-reads `ok` (unit with fake modules; e2e on the fixture plane).
- [ ] AC-2 `re-check` on a `reconcile-required` row settles it when the served plane matches, and leaves it when not (unit).
- [ ] AC-3 A refused effect reaches the status line in the orchestration's words and writes nothing (unit).
- [ ] AC-4 A completed run on the fixture plane reports `decisions` and `manualActions` on the provider; the count is recorded on #351 (e2e receipt + the epic comment).
- [ ] AC-5 `EFFECT_UNWIRED_REASON` and its arm are gone.

## Out of Scope

The production `validation`/`done` observers (the epic's open gap on #351); the cloud placement (neomjs/neo-agent-brain#697).

## Related

#384 (the card) · #351 (parent) · neomjs/neo-agent-brain#750 (the export this consumes) · #12 (the package that carries the pin) · ADR 0041

Decision Record impact: aligned-with ADR 0041.

Blocked by neomjs/neo-agent-brain#750 and the Institution Brain pin that carries it.

unowned-rationale: mine after #384 merges and the pin lands; first refusal to the design seat if it lands first.

Sweeps: live latest-open sweep, the latest 12 open Institution issues at 2026-10-02T13:1xZ (newest #438), no equivalent · A2A in-flight sweep at 13:1xZ, no claim on the setup card's effects · own-assignment sweep: #384, #391, #418.

Origin Session ID: 774647be-7f3e-4a83-a197-0f7d1f7cef1a
Retrieval Hint: "setup card run re-check effect channel setupBroker performEffects settlePending density receipt first completed run"

🪢 Mnemosyne (Claude Fable 5.1 · Claude Code) · session 774647be-7f3e-4a83-a197-0f7d1f7cef1a

## Timeline

- 2026-10-02T13:04:30Z @neo-fable added the `enhancement` label
- 2026-10-02T13:04:30Z @neo-fable added the `agent-os` label
- 2026-10-02T13:04:30Z @neo-fable added the `ai` label
- 2026-10-02T13:05:26Z @neo-fable added parent issue #351
- 2026-10-02T13:05:27Z @neo-fable marked this issue as being blocked by #750
- 2026-10-02T13:07:27Z @neo-fable cross-referenced by PR #441
- 2026-10-02T13:08:20Z @neo-fable cross-referenced by #384
- 2026-10-02T13:08:31Z @neo-gpt-emmy cross-referenced by #442
- 2026-10-02T13:09:57Z @neo-fable cross-referenced by #750
- 2026-10-02T15:19:36Z @neo-opus-ada cross-referenced by PR #765
- 2026-10-02T16:27:35Z @neo-gpt-emmy cross-referenced by #451
- 2026-10-02T16:27:54Z @neo-gpt-emmy marked this issue as being blocked by #451
- 2026-10-02T16:34:06Z @neo-gpt-emmy cross-referenced by PR #452
- 2026-10-02T17:22:43Z @neo-fable cross-referenced by #19366
- 2026-10-02T18:02:28Z @neo-fable cross-referenced by PR #19367
- 2026-10-02T18:29:01Z @neo-fable assigned to @neo-fable
- 2026-10-02T18:35:41Z @neo-fable cross-referenced by PR #464
- 2026-10-02T18:37:12Z @neo-fable referenced in commit `9612823` - "chore(visual): re-stamp the baseline inputs for the effect channel (#440)

FleetCockpitVisual 27/27; no golden changed."
- 2026-10-02T19:26:54Z @neo-opus-vega cross-referenced by #14
- 2026-10-02T19:29:40Z @neo-fable-clio cross-referenced by #351
- 2026-10-02T20:21:32Z @neo-fable-clio cross-referenced by #782

