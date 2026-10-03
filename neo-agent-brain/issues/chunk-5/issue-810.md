---
id: 810
title: A consent changed after an accepted effect re-applies it as a new input
state: OPEN
labels:
  - bug
  - ai
  - architecture
  - agent-os
assignees:
  - neo-fable
createdAt: '2026-10-03T12:25:26Z'
updatedAt: '2026-10-03T12:25:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/810'
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
# A consent changed after an accepted effect re-applies it as a new input

## Context

Measured by @neo-fable on 2026-10-03 (node probe over the branch's modules, temp layout, recording runner, production carrier + secret observers, the served plane scripted as the target's) while fixing #786/#790:

1. `local-small` + credential consented, `performEffects` → three accepted effects, every row `ok`.
2. The operator changes the preset to `hosted` and gives a provider key.
3. Evaluate: `write-env: ok (observed; matches the accepted receipt)`, `compose-up: ok`; with #790 the secrets row turns `failed` (the hosted key file is missing) and the next run writes the set.
4. Another run: the carrier is NOT rewritten (no `NEO_GEMINI_API_KEY_FILE`, still local-small's env); docker calls stay at one.

The card shows a finished run on a preset the operator no longer chose; `validation` (#796) may catch the dimension later; no row offers a way to apply the change. Filed by the planner under the filing freeze; the shape was settled between the recipe's author and the finder at 09:20Z (fork (a)); the finder builds it.

## The Problem

ADR 0041 says a step's status is a fresh observation for the bound target and the recipe version (§2.3), that an accepted effect is never re-run because a renderer resumed (§2.6), and that a change of target or recipe version retires every proof (§2.7). It says nothing about a change of **consent**. The implementation fills the gap wrongly: `evaluateEffect` (`ai/services/fleet/firstRunRecipe.mjs:221`) compares the accepted receipt's digest with the file on the host, never the receipt's **input** with what the CURRENT consents would render; and `compose-up`'s input holds no carrier digest, so even a rewritten carrier would not be a new application of it. A receipt therefore stays proof for an intent the operator has abandoned — the green-over-stale machine the ADR exists to kill, one layer up.

Worse with #796 in place: after a preset change the `validation` observer probes the NEW preset's provider from the host and may pass, the served plane is healthy, the recorded witness still reads recalled — and `done` reads `ok` on a plane running the OLD preset.

## The Architectural Reality

- `firstRunRecipe.mjs`: `evaluateEffect` reads `receipt.outcome`, `receipt.digest` vs `observed.digest`; `staleGate` / `evaluateDone` gate on served-plane + validation freshness (#796).
- `setupOrchestration.mjs`: `EFFECT_ORDER = [writeSecrets, writeEnv, composeUp, verify]`; `inputs[write-env]` renders the carrier from the consents; `HOST_FILE_EFFECTS` settle by their own observation + the pending receipt's expected digest (#790).
- `hostEffects.mjs` / `setupRunRecord.mjs`: receipts carry `{effectId, outcome, digest, at, …}`; `retireCurrentProof` moves proofs into history on target/version change.
- `verifyEffect.mjs` (#796): one write per attempt, keyed by `{runId, planeId, marker}`; a `newAttempt` is explicit consent for the SAME composition.

## The Fix (shape (a), settled)

**Every effect's at-most-once guard is keyed by its input.** A receipt records the input it was accepted for; the evaluator renders the effect's CURRENT input from the consents and the upstream receipts; an effect whose current input differs from its accepted receipt's input reads `pending` with the reason "accepted for an earlier input (…); the consents changed", and `performEffects` applies it again as a NEW input — never a replay of the old one.

1. `write-env`: input = the rendered carrier (digest of the rendering from the current consents, through the one rendering both the orchestration and the observer call — move it where both can). A changed rendering → pending → rewritten.
2. `write-secrets`: input = the consented preset's whole secret set (the #790 observer already reads the whole set; its receipt records the set's digest).
3. `compose-up`: input gains the carrier's digest, so a rewritten carrier restarts the plane; the status line names the consequence in words ("the plane restarts").
4. `verify`: input gains the compose-up receipt it followed. A new composition → the verification section reads "witnessed the earlier composition" → pending → a new attempt as a new input (one write per attempt holds; `newAttempt` stays the explicit second write for the SAME composition).
5. ADR 0041 amendment, its own docs PR merged first (ADR 0005 §6.5): §2.6 — "an accepted effect is never re-run for the same input; a changed input is a new effect, never a replay"; §2.7 — a consent change retires nothing wholesale: each receipt stays proof only for the input it recorded. Status-row line in the ADR table.

## Acceptance Criteria

- [ ] AC-1 The four-step probe above, as a unit arm: after the preset change, `write-env`, `write-secrets` and `compose-up` read `pending` with the "earlier input" reason; after `performEffects`, the carrier carries the hosted preset's env, the secret set is complete, docker was called a second time, and the rows read `ok` for the NEW inputs.
- [ ] AC-2 `verify` after a new composition reads pending with its reason and performs ONE new attempt with a fresh marker; the earlier attempt stays in the section's history; a resume without a consent change still writes nothing (the #796 arm stays green).
- [ ] AC-3 A consent change that leaves an effect's rendering identical keeps that effect `ok` (no wholesale retire): unit arm with a preset change that touches the carrier but not the secret set.
- [ ] AC-4 The ADR docs PR (§2.6 / §2.7 sentences + status row) merges ahead of the implementation PR; the implementation PR's body names it.
- [ ] AC-5 Live receipt: on the maintainer plane, change the consented preset after a complete run and watch the rows turn pending, the re-run rewrite the carrier and restart the plane, and `done` read `ok` only afterwards (the first-run CLI's own output, pasted on this ticket).

## Out of Scope

- The card's fresh-run exit (#475, Institution) — a different door for a different problem (a run that cannot settle).
- Refusing consent changes after acceptance (fork (c)): rejected — with (a) a consent change is an ordinary, honestly reported operator action.

## Avoided Traps

- Fork (b), retire every proof on any consent change: coarser than the truth and re-runs effects whose input did not change.
- A `try/catch` around the carrier comparison or a "force" flag: both hide the input mismatch instead of naming it.

## Related

Institution #351 (parent epic), #786 / #790 (host-file effects settle by their own observation), #796 (verify + validation gates), #785 (observer context), ADR 0041 (`learn/agentos/decisions/0041-bootstrap-record-verified-plane-handoff.md`), Institution #475 (the fresh-run exit).

Decision Record impact: amends ADR 0041 (§2.6 / §2.7, one sentence each; docs PR first per ADR 0005 §6.5).

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-10-03 12:24Z and a keyword search ("preset changed receipt") — no equivalent. A2A in-flight claim sweep: the only claim is the finder's proposal to build (12:21Z); no competing claim. Memory Core rationale sweep: the 09:20Z design answer (fork (a) chosen; (b) and (c) rejected with reasons) is the prior decision. Own-assignment sweep: #782 (closed by #796) was the neighbour; nothing of mine owns it. Structure map: `npm run ai:structure-map -- --files --loc` → `ai/services/fleet` (siblings `firstRunRecipe.mjs`, `hostEffects.mjs`, `setupOrchestration.mjs`, `setupRunRecord.mjs`) — no new file expected.

handoff: @neo-fable (the finder, who measured it and builds both PRs; assigned).

Retrieval Hint: "input-keyed at-most-once receipt proves the input it recorded consent change re-applies write-env compose-up carrier digest verify new composition"

Origin Session ID: 0f761cf3-1901-468a-8bc7-e4eb6f0929cb

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 0f761cf3-1901-468a-8bc7-e4eb6f0929cb

## Timeline

- 2026-10-03T12:25:27Z @neo-fable-clio assigned to @neo-fable
- 2026-10-03T12:25:27Z @neo-fable-clio added the `bug` label
- 2026-10-03T12:25:27Z @neo-fable-clio added the `ai` label
- 2026-10-03T12:25:28Z @neo-fable-clio added the `architecture` label
- 2026-10-03T12:25:28Z @neo-fable-clio added the `agent-os` label
- 2026-10-03T12:26:05Z @neo-fable-clio added parent issue #351

