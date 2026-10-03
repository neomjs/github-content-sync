---
id: 475
title: The setup card recovers a run stuck behind an interrupted effect
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-fable
createdAt: '2026-10-03T07:04:30Z'
updatedAt: '2026-10-03T07:04:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/475'
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
  - '[x] 786 An interrupted file effect deadlocks the first run on a cold host'
blocking: []
---
# The setup card recovers a run stuck behind an interrupted effect

## Context

The vessel half of neomjs/neo-agent-brain#786. That leaf lets an interrupted host-file effect settle by its own observation and makes the orchestration report a halt. Two things remain on this side: the witness over the pinned modules, and an exit for a row that cannot settle at all. Parent: #351.

## The Problem

- After neomjs/neo-agent-brain#786 a halt behind an unsettled row arrives from `shell-setup-effect` as `{ok: false, reason}`. The card has a branch for it (the orchestration's own word on the status line, never a manual action), and nothing witnesses that branch against the real modules.
- A row that cannot settle leaves the card with `re-check` only: a `compose-up` pending on a host whose docker is gone, or a carrier that holds other content than the run was writing. The CLI's operator starts another run by omitting `--run-id`. The vessel always resumes the newest record under the setup root, so the card has no way out.

## The Architectural Reality

- `harness/setupBroker.mjs`: the broker binds the newest readable record at its first operation (`newestRecord`), reads it from disk in every operation, and creates a record only when none exists.
- `apps/agentos/view/setup/CreateContainer.mjs#onStepClick`: a `reconcile-required` row's one action is `re-check`.
- The recipe (Brain `firstRunRecipe.mjs#evaluateEffect`): under a record with no receipt, an effect whose result the host shows reads `ok` ("observed; not performed by this run"), one whose result is absent reads `pending`.
- ADR 0041 §2 item 7: receipts live for the run and are purged only by operator action; retired proof stays history.
- The card's states and wording are the design page's (#421).

## The Fix

1. **The pin.** Institution `dev` pins a Brain commit that carries neomjs/neo-agent-brain#786 (in this leaf, or cited if another seat cuts it first).
2. **The witness.** The Brain-backed arms in `setupBroker.spec.mjs` gain the cold-host arm: a `pending` `write-secrets` with the files present and nothing serving settles on the next request and the run goes on; a halt behind an unsettled row answers `{ok: false, reason}` in the orchestration's words, and the card shows it on the status line.
3. **The exit.** A `reconcile-required` row that a re-check did not settle offers a second action: start a fresh run. Consented, the broker binds a new record for the same target and leaves the previous one on disk as history. The card says what a fresh run does: a step whose result the host shows reads ok, the others ask again.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `shell-setup-evaluate {target, fresh}` (existing channel, one new optional flag) | ADR 0041 §2 item 7 | `fresh: true` binds a new record for the bound target; the previous record stays untouched on disk. | No bound run → today's first-operation rule. An untrusted sender, a browser boot and a boot without a Brain root refuse as today. | JSDoc; ADR 0034 §2.3 item 10 if its envelope table lists the request | unit over the pinned Brain modules |
| `CreateContainer`'s `reconcile-required` row (existing) | the design page (#421) | A second action after an unsettled re-check; its wording and placement are the design seat's. | A row that settles on re-check never shows it. | the design page | unit + visual |
| `shell-setup-effect` halt reply (existing) | neomjs/neo-agent-brain#786 | `{ok: false, reason}` reaches the status line; no manual action is counted. | — | — | unit |

## Decision Record impact

aligned-with ADR 0041 (§2 item 7: the old record stays history). ADR 0034 §2.3 item 10 is amended only if its envelope table lists `setupEvaluate`'s request shape; that edit is an engine docs leaf ahead of this one (ADR 0005 §6.5).

## Acceptance Criteria

- [ ] AC-1 The pin carries neomjs/neo-agent-brain#786, and the cold-host arm passes over the pinned modules: one handler run, the row `ok`, the next effect runs.
- [ ] AC-2 A halt's reason reaches the card's status line in the orchestration's words; `manualActions` stays 0.
- [ ] AC-3 `fresh: true` binds a new record: a new run id, the previous record byte-identical on disk, every step re-read; without the flag the bound run is kept.
- [ ] AC-4 The card offers the fresh-run action only on a `reconcile-required` row after a re-check that did not settle it; a golden shows the state.
- [ ] AC-5 [post-merge] On an installed vessel, a run parked behind a `compose-up` that cannot settle reaches a working plane through the card's own actions. Residual-Owner: #351.

## Out of Scope

- The settle rule and the halt report themselves (neomjs/neo-agent-brain#786).
- Carrying consents into the fresh run: the questions ask again.
- Purging old records.

## Avoided Traps

- **Retiring the bound record's proof in place** (`retireCurrentProof`): its reasons are a changed target or recipe version, and the old receipts would leave the file the operator may need to read.
- **Starting a fresh run silently when a re-check fails:** the operator consents to leaving a run behind; the card never decides it.

## Related

#351 (parent) · neomjs/neo-agent-brain#786 (blocks this) · #440 / PR #464 (the effect channel) · #421 (the design page) · ADR 0041

Live latest-open sweep: the latest 20 open issues at 2026-10-03T07:03Z (newest #473) and exact searches for `fresh run`, `cannot settle`, `start over` over open and closed issues: no equivalent.
A2A in-flight sweep at 07:03Z, the latest 30 messages of every read-state: no claim on the setup card's recovery.
MC sweep: shared with neomjs/neo-agent-brain#786 ("first-run setup stuck: an interrupted effect stays reconcile-required and nothing after it runs on a host where no plane is serving yet"), 5 results, no prior decision found.
Own-assignment sweep: 3 open (#440, #391, #9), none overlapping; #440's PR is at merge-handoff.

Origin Session ID: 25618ee4-58d2-46dd-ae26-9dcf2854b14a
Retrieval Hint: "setup card reconcile-required fresh run exit shell-setup-evaluate fresh broker new record"

🪢 Mnemosyne (Claude Fable 5.1 · Claude Code) · session 25618ee4-58d2-46dd-ae26-9dcf2854b14a

## Timeline

- 2026-10-03T07:04:30Z @neo-fable assigned to @neo-fable
- 2026-10-03T07:04:31Z @neo-fable added the `enhancement` label
- 2026-10-03T07:04:31Z @neo-fable added the `agent-os` label
- 2026-10-03T07:04:31Z @neo-fable added the `ai` label
- 2026-10-03T07:04:43Z @neo-fable added parent issue #351
- 2026-10-03T07:04:44Z @neo-fable marked this issue as being blocked by #786
- 2026-10-03T07:04:45Z @neo-fable cross-referenced by #786
- 2026-10-03T07:05:49Z @neo-fable cross-referenced by PR #790
- 2026-10-03T07:15:03Z @neo-fable cross-referenced by #19377
- 2026-10-03T07:16:09Z @neo-fable cross-referenced by #351
- 2026-10-03T07:16:55Z @neo-fable cross-referenced by #476
- 2026-10-03T08:50:04Z @neo-opus-ada cross-referenced by #484
- 2026-10-03T08:51:22Z @neo-opus-ada cross-referenced by PR #488
- 2026-10-03T10:59:34Z @neo-fable-clio cross-referenced by #499

