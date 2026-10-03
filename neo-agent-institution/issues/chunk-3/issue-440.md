---
id: 440
title: 'The setup card''s run and re-check actions reach the vessel''s effect channel, and the first completed run records its density'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-fable
createdAt: '2026-10-02T13:04:28Z'
updatedAt: '2026-10-03T07:16:10Z'
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
closedAt: '2026-10-03T07:13:39Z'
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
| `shell-setup-effect {effectId}` (existing channel) | neomjs/neo-agent-brain#750's `performEffects` | Runs the one consented effect in the CLI's order rules; replies the re-evaluated run. | A refusal (preset env, credentials) → `{ok: false, reason}` in the orchestration's words; a `reconcile-required` row on it or ahead of it → halted: the reply is the evaluation with that row unchanged, never replayed. | JSDoc | `setupBroker.spec.mjs`: one-effect run · refusal · the halt |
| `shell-setup-evaluate` (existing) | #750's `settlePending` | Settles interrupted effects by a matching observation before evaluating. | Nothing to settle → unchanged. | JSDoc | `setupBroker.spec.mjs` resume arm |
| `CreateContainer#runEffect` (existing) | #384 | A shell that cannot run effects at all (`no-shell`, `not-packaged`, `no-brain-root`) → the row's action is the operator's instruction (the CLI's command), counted as a manual action. | A refusal from the orchestration (the preset's env set, the credential composition) or a run that failed (`<effect> could not run: …`, an unreadable record included) → the status line in the shell's words, never a manual action. | JSDoc | `createContainer.spec.mjs`: the shell-unavailable arm · the orchestration-refusal arm |
| `setupBroker` record operations (existing) | ADR 0041 §2.6 · the orchestration's one-writer precondition | Every serialized operation reads the bound run's record from disk; the broker keeps the record's path only. An effect whose acknowledgement write was rejected is found `pending` by the next request and settled by observation, never run again. | A bound record that is gone or unreadable → the operation is refused by name; nothing is written and no fresh run takes its place. | JSDoc | `setupBroker.spec.mjs` over the Brain's own modules: the rejected-acknowledgement arm (same broker · fresh broker) · the consent arm · the accepted control · the unreadable-record arm |

## Acceptance Criteria

- [x] AC-1 With a Brain pin carrying neomjs/neo-agent-brain#750, `run` on `write-env` writes the carrier through the host-effect module and the row re-reads `ok` (unit with fake modules; e2e on the fixture plane).
- [x] AC-2 `re-check` on a `reconcile-required` row settles it when the served plane matches, and leaves it when not (unit).
- [x] AC-3 A refused effect reaches the status line in the orchestration's words and writes nothing (unit).
- [x] AC-4 A completed run on the fixture shell reports `decisions` and `manualActions` on the mounted provider (the e2e reads the provider: 6 / 0); the count is recorded on #351 as the fixture path's projection — the fixture shell assigns the step statuses, so it witnesses the consumer, never a host run.
- [x] AC-5 `EFFECT_UNWIRED_REASON` and its arm are gone.

## Evidence boundary

- **Here (L2 + consumer e2e):** the broker over module doubles and over the Brain's own modules on a temp layout; the mounted card against the fixture shell.
- **L3-deferred — owner #351:** a completed run on an installed host (three effects through the vessel on a Docker host) and that run's density. No receipt on this ticket claims it.
- **Documentation obligation — owner #351 until its leaf exists:** ADR 0034 §2.3 item 10 still says the effect handler refuses every call; an engine docs leaf corrects that sentence when this leaf's PR merges.

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
- 2026-10-03T06:39:43Z @neo-fable referenced in commit `5ffda9a` - "chore(merge): bring origin/dev into the branch (#440)"
- 2026-10-03T06:39:44Z @neo-fable referenced in commit `6a1c792` - "fix(harness): every setup record operation starts from the record on disk, so an effect whose acknowledgement write was rejected is never replayed (#440)

The broker kept the run's record in memory and re-read it only on a fresh boot. A host that rejected the write acknowledging an effect left the pending receipt on disk and an older record in the broker: the next request ran the handler again, and the next consent wrote the guard away. The broker now holds the record's path only; every serialized operation reads the bound record from disk, a bound record that is gone or unreadable refuses the operation, and the writer's returned record is the working copy inside one operation.

The regression arms run the broker over the pinned Brain's own recipe, record, host-effect and orchestration modules on a temp layout: the rejected-acknowledgement arm (same broker, fresh broker), the consent arm, the accepted control and the unreadable-record arm. The fixture e2e reads the completed run's density from the mounted provider (6 decisions, 0 manual actions). The Create door no longer matches the retired unwired reason."
- 2026-10-03T07:04:31Z @neo-fable cross-referenced by #475
- 2026-10-03T07:13:39Z @tobiu referenced in commit `424fa0e` - "feat(agentos): the setup card's run and re-check reach the vessel's effect channel through the shared orchestration (#440) (#464)

* feat(agentos): the setup card's run and re-check reach the vessel's effect channel through the shared orchestration (#440)

shell-setup-effect runs the one consented effect through the Brain's setupOrchestration (the settle pass first, effectIds in the recipe's order rules, a refusal in the orchestration's words with nothing written) inside the serialized chain, the config source injected by main; evaluate settles an interrupted effect before reading. The card counts a manual action only when the shell cannot run effects at all; an orchestration refusal is the shell's own word. The card relays the Create door's first persistence to its owner. The e2e fixture runs the three effects to a completed run.

* chore(visual): re-stamp the baseline inputs for the effect channel (#440)

FleetCockpitVisual 27/27; no golden changed.

* fix(harness): every setup record operation starts from the record on disk, so an effect whose acknowledgement write was rejected is never replayed (#440)

The broker kept the run's record in memory and re-read it only on a fresh boot. A host that rejected the write acknowledging an effect left the pending receipt on disk and an older record in the broker: the next request ran the handler again, and the next consent wrote the guard away. The broker now holds the record's path only; every serialized operation reads the bound record from disk, a bound record that is gone or unreadable refuses the operation, and the writer's returned record is the working copy inside one operation.

The regression arms run the broker over the pinned Brain's own recipe, record, host-effect and orchestration modules on a temp layout: the rejected-acknowledgement arm (same broker, fresh broker), the consent arm, the accepted control and the unreadable-record arm. The fixture e2e reads the completed run's density from the mounted provider (6 decisions, 0 manual actions). The Create door no longer matches the retired unwired reason."
- 2026-10-03T07:13:39Z @tobiu closed this issue
- 2026-10-03T07:15:03Z @neo-fable cross-referenced by #19377
- 2026-10-03T07:24:28Z @neo-fable-clio cross-referenced by PR #796
- 2026-10-03T08:26:26Z @neo-fable-clio cross-referenced by #481

