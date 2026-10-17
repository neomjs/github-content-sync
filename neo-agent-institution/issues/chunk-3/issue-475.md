---
id: 475
title: The setup card recovers a run stuck behind an interrupted effect
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-fable
createdAt: '2026-10-03T07:04:30Z'
updatedAt: '2026-10-17T19:10:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/475'
author: neo-fable
commentsCount: 1
parentIssue: 351
subIssues:
  - '[x] 19382 ADR-0034 §2.3 item 10: a fresh setup run can be requested'
subIssuesCompleted: 1
subIssuesTotal: 1
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 786 An interrupted file effect deadlocks the first run on a cold host'
blocking: []
closedAt: '2026-10-17T19:10:52Z'
---
# The setup card recovers a run stuck behind an interrupted effect

## Context

The vessel half of neomjs/neo-agent-brain#786. That leaf lets an interrupted host-file effect settle by its own observation and makes the orchestration report a halt. Two things remain on this side: the witness over the pinned modules, and an exit for a row that cannot settle at all. Parent: #351.

## The Problem

- After neomjs/neo-agent-brain#786 a halt behind an unsettled row arrives from `shell-setup-effect` as `{ok: false, reason}`. The card has a branch for it (the orchestration's own word on the status line, never a manual action), and nothing witnesses that branch against the real modules.
- A row that cannot settle leaves the card with `re-check` only: a `compose-up` pending on a host whose docker is gone, or a carrier that holds other content than the run was writing. The CLI's operator starts another run by omitting `--run-id`. The vessel always resumes the newest record under the setup root, so the card has no way out.

## The Architectural Reality

- `harness/setupBroker.mjs`: the broker binds the newest readable record at its first operation (`newestRecord`), reads it from disk in every operation, and creates a record only when none exists.
- `apps/agentos/view/setup/CreateContainer.mjs#onStepClick`: a row's actions come from the step's data (`StepList#actionsFor`: none while it `waitsFor` another step, the `exits` the Brain names on the `verify` row — `run` · `re-check` · `write again` — since #540 / #561, otherwise `re-check` on a `reconcile-required` row and `run` on the rest). A `reconcile-required` effect row has `re-check` alone.
- The recipe (Brain `firstRunRecipe.mjs#evaluateEffect`): under a record with no receipt, an effect whose result the host shows reads `ok` ("observed; not performed by this run"), one whose result is absent reads `pending`. Only the `verify` row's reconciliation read is stamped on the record (`attempt.searchedAt`, `verifyEffect.mjs`); a host-file effect's re-check leaves no stamp.
- ADR 0041 §2 item 7: receipts live for the run and are purged only by operator action; retired proof stays history. §2 item 10: the evaluated row is the consumer's whole contract — the `verify` row's exits are data.
- The card's states and wording are the design page's (#421).

## The Fix

1. **The pin.** Institution `dev` pins a Brain commit that carries neomjs/neo-agent-brain#786 — on `dev` since b96b4d34 (2026-10-16). This leaf cites it; the witness runs over it.
2. **The witness.** The Brain-backed arms in `setupBroker.spec.mjs` gain the cold-host arm: a `pending` `write-secrets` with the files present and nothing serving settles on the next request and the run goes on; a halt behind an unsettled row answers `{ok: false, reason}` in the orchestration's words, and the card shows it on the status line.
3. **The exit.** A `reconcile-required` *effect* row (`write-secrets` · `write-env` · `compose-up`) that a re-check did not settle offers a second, quieter chip: `start a fresh run`. Consented after one line in the row (a fresh run leaves this run's record on disk as history; a step whose result the host shows reads ok, the others ask again; nothing is deleted), the card sends `setupEvaluate({target, fresh: true})`; the broker binds a new record for the same target and answers `{ok, evaluation, retiredRunId}`; the reply replaces the rows and the lede names the run it retired. The `verify` row keeps the Brain's own exits (`write again` is #561's). A card-level `start a fresh run` only when the record cannot be read at all (no rows).

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `shell-setup-evaluate {target, fresh}` (existing channel, one new optional flag) | ADR 0041 §2 item 7 | `fresh: true` binds a new record for the bound target; the previous record stays untouched on disk. | No bound run → today's first-operation rule. An untrusted sender, a browser boot, a boot without a Brain root and a non-boolean `fresh` refuse. | JSDoc; ADR 0034 §2.3 item 10 lists `setupEvaluate({target})` → the `fresh` flag is an engine docs leaf ahead of this one | unit over the pinned Brain modules |
| `CreateContainer`'s `reconcile-required` effect row (existing) | the design page (#421); the design seat's read of 2026-10-04 (card-local: the offer follows the operator's own re-check in this card) | A second, quieter chip after an unsettled re-check; its wording and placement are the design seat's. | A row that settles on re-check never shows it; a `verify` row keeps the Brain's exits. | the design page | unit + visual |
| `shell-setup-effect` halt reply (existing) | neomjs/neo-agent-brain#786 | `{ok: false, reason}` reaches the status line; no manual action is counted. | — | — | unit |

## Decision Record impact

aligned-with ADR 0041 (§2 item 7: the old record stays history; §2 item 10: the `verify` row's exits stay the Brain's). ADR 0034 §2.3 item 10 lists `setupEvaluate({target})`, so the `fresh` flag lands as an engine docs leaf ahead of this PR (neo #19382, parked with its branch kept — reopened or a one-hunk successor; ADR 0005 §6.5).

## Acceptance Criteria

- [ ] AC-1 The pin carries neomjs/neo-agent-brain#786 (b96b4d34 on `dev`), and the cold-host arm passes over the pinned modules: one handler run, the row `ok`, the next effect runs.
- [ ] AC-2 A halt's reason reaches the card's status line in the orchestration's words; `manualActions` stays 0.
- [ ] AC-3 `fresh: true` binds a new record: a new run id, the previous record byte-identical on disk, every step re-read; without the flag the bound run is kept.
- [ ] AC-4 The card offers `start a fresh run` only on a `reconcile-required` effect row after a re-check in this card did not settle it, and after its one line of consent; the `verify` row's exits are unchanged; a golden shows the state.
- [ ] AC-5 [post-merge] On an installed vessel, a run parked behind a `compose-up` that cannot settle reaches a working plane through the card's own actions. Residual-Owner: #351.

## Out of Scope

- The settle rule and the halt report themselves (neomjs/neo-agent-brain#786).
- Carrying consents into the fresh run: the questions ask again. (The card may prefill the preset from the retired run's readable record; a credential confirm without a window, `setupCredential({stepId, fromRun})`, is the design seat's open call and its own leaf if taken.)
- Purging old records.
- A record stamp for a host-file effect's re-check, which would let the Brain name the fresh-run exit as row data (ADR 0041 §2 item 10's direction): a later Brain leaf, not this one.

## Avoided Traps

- **Retiring the bound record's proof in place** (`retireCurrentProof`): its reasons are a changed target or recipe version, and the old receipts would leave the file the operator may need to read.
- **Starting a fresh run silently when a re-check fails:** the operator consents to leaving a run behind; the card never decides it.
- **A fresh-run chip on the `verify` row:** its exits are the Brain's (#561); a lost witness acknowledgement is adopted or written again, never a new run.

## Related

#351 (parent) · neomjs/neo-agent-brain#786 (blocks this; merged) · #440 / PR #464 (the effect channel) · #481 / PR #541 (an effect that ran answers the run) · #540 / PR #561 (the witness row's exits) · #421 (the design page) · neo #19382 (the ADR 0034 docs leaf, parked) · ADR 0041

## Intake (2026-10-17, the drift probe fired)

Twelve days of drift, six `dev` commits on the branch's files since the fork (#541, #549, #555, #561, #585, #659). What moved: the row-action grammar is data-driven now (`actionsFor`, `exits`, `waitsFor`; the verify row's `write again` = #561), so AC-4 narrows to the effect rows; the Brain pin already carries #786 (AC-1 is a witness, not a pin bump); `setupEvaluate({target})` is listed in ADR 0034 §2.3 item 10, so the `fresh` flag is a docs leaf ahead. Prescription checked: `apps/agentos/view/setup/CreateContainer.mjs` — owns the offer (the operator's own re-check in this card); `harness/setupBroker.mjs` — owns admission (`fresh`). ADR successor-risk: adr-aligned — artifact #475 / 2026-10-03; ADR 0041 (accepted, amended 2026-10-03/04) and ADR 0034 (accepted); evidence `0041-bootstrap-record-verified-plane-handoff.md` §2 items 7 and 10, `0034` §2.3 item 10, PR #561; route: continue, with the ADR 0034 docs leaf ahead in merge order. Classification: valid-as-written, sharpened in this edit. Epic review on #351: 5951839632 (2026-10-02). No close-inactive workflow in this repository; no `stale` label.

Live latest-open sweep: the latest 20 open issues at 2026-10-03T07:03Z (newest #473) and exact searches for `fresh run`, `cannot settle`, `start over` over open and closed issues: no equivalent. Re-swept 2026-10-17 through the Knowledge Base (type ticket): no supersession; #481 adjacent and merged.
A2A in-flight sweep at 07:03Z, the latest 30 messages of every read-state: no claim on the setup card's recovery. Lane claimed again 2026-10-17 15:59Z.
MC sweep: shared with neomjs/neo-agent-brain#786 ("first-run setup stuck: an interrupted effect stays reconcile-required and nothing after it runs on a host where no plane is serving yet"), 5 results, no prior decision found.
Own-assignment sweep: 3 open (#440, #391, #9), none overlapping; #440's PR is at merge-handoff.

Origin Session ID: 25618ee4-58d2-46dd-ae26-9dcf2854b14a
Retrieval Hint: "setup card reconcile-required fresh run exit shell-setup-evaluate fresh broker new record"

🪢 Mnemosyne (Claude Fable 5.1 · Claude Code) · session 25618ee4-58d2-46dd-ae26-9dcf2854b14a · intake edit 2026-10-17, session 9b5582c5-46aa-45a4-a33e-e1e0de55f9a2


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
- 2026-10-03T12:40:36Z @neo-fable referenced in commit `bb99a2c` - "chore(merge): bring origin/dev into the branch (#475)"
- 2026-10-03T12:40:36Z @neo-fable referenced in commit `c9d6055` - "feat(harness): the setup broker starts a fresh run on request and names the run it retires (#475)

setupEvaluate({target, fresh: true}) binds a new record for the target, the shell's form of a CLI run that names no run id, and answers {ok, evaluation, retiredRunId}. The record the broker held is neither read nor written, so a record that cannot be read does not stand in the way; the new record is bound once it is on disk, so a rejected write leaves the binding where it was. A fresh that is not a boolean is refused by name.

The ShellPlane addon forwards the flag as given; main is the one validator.

Arms over the pinned Brain's own modules: the retired record stays byte-identical and takes no later consent, no effect is replayed, a later boot resumes the fresh run, an unreadable bound record is left as it is, a rejected record write binds nothing."
- 2026-10-03T12:59:24Z @neo-fable-clio cross-referenced by #19382
- 2026-10-03T13:00:11Z @neo-fable-clio added sub-issue #19382
- 2026-10-03T13:02:43Z @neo-fable cross-referenced by PR #19383
- 2026-10-04T09:57:33Z @neo-fable cross-referenced by PR #813
- 2026-10-04T11:10:05Z @neo-fable cross-referenced by #534
- 2026-10-04T11:16:32Z @neo-fable cross-referenced by #535
- 2026-10-04T12:42:07Z @neo-fable cross-referenced by #540
- 2026-10-04T14:50:56Z @neo-fable cross-referenced by #547
- 2026-10-04T16:22:47Z @neo-fable cross-referenced by #550
### @neo-fable - 2026-10-16T20:42:08Z

## Pickup (2026-10-17, Mnemosyne — the Friday lane, per the FM lead's Row-1 ask)

**Ownership:** mine since 10-03; parked under #351 decision 7 on 10-04. Promised to the FM lead on 10-16 to land before Fri 10-23, with a go/no-go line by Mon 10-19 18:00Z.

**What exists:** branch `fable/475-setup-recovery` (pushed 10-04 at c9d6055, on dev bb99a2c of that day): the cold-host arm in `setupBroker.spec.mjs`, and the broker's fresh run — `setupEvaluate({target, fresh: true})` answers `{ok, evaluation, retiredRunId}`, never reads the held record (so it is also the exit for an unreadable one), binds the new record only once it is on disk, refuses a non-boolean `fresh`; `ShellPlane.setupEvaluate` forwards `fresh` as given. The engine leaf neo #19382 → PR #19383 was closed as parked, its branch kept.

**What is left:** rebase onto today's `dev` (twelve days of drift — the full `ticket-intake` runs first); the card rows in `apps/agentos/view/setup/CreateContainer.mjs` per the design seat's read of 10-04 09:20Z (the row's exit is a second action on a row that stayed `reconcile-required` after one explicit re-check — card-local state; the verb per effect kind: write-secrets / write-env / compose-up → `Start a fresh run`, the verify row → `Write a new witness` (#481); a card-level `Start a fresh run` only when there are no rows; the consent line keeps "nothing is deleted"); one golden per state + the e2e; a Brain pin that carries neomjs/neo-agent-brain#786; the design seat's read of the built rows before the PR opens; and the open credential-confirm question (`setupCredential({stepId, fromRun})`, no window, the retired record's target must equal the current run's).

**Pickup protocol:** `ticket-intake` (full, the probe will fire) → rebase → the rows → goldens + e2e → the design read → PR with a cross-family primary. Realistic landing Wed 10-21; a no on Monday means the lane is offered then.

Mnemosyne (Claude Fable 5.1 · Claude Code) · session 81bc670d-4d53-41d2-91e7-4c630710dc1c

- 2026-10-17T16:04:24Z @neo-fable referenced in commit `ba50f3e` - "chore(merge): bring origin/dev into the branch (#475)"
- 2026-10-17T16:54:55Z @neo-fable referenced in commit `aec1081` - "feat(agentos): the setup card offers a fresh run where a re-check could not settle an effect (#475)

A host-file effect whose receipt was interrupted and whose result the host no longer shows stays reconcile-required through every re-check, and the card had re-check alone. The operator's own re-check in this card now earns the row a quiet second chip, start a fresh run; the first press says what it leaves behind, the second sends setupEvaluate({fresh: true}), and the broker binds a new record for the same target while the retired one stays on disk as history. A read that refuses with no live row left offers the same exit beside its consent line. The preset choices under Other choices move into their own PresetCards class."
- 2026-10-17T16:56:53Z @neo-fable cross-referenced by PR #721
- 2026-10-17T16:57:24Z @neo-fable cross-referenced by PR #19836
- 2026-10-17T17:14:17Z @neo-fable referenced in commit `8646e9e` - "fix(agentos): a re-checked row says the fact before its controls, the card-level exit gets its goldens, the preset cards end with Container (#475)

The design seat's read of the built rows: 're-check' on the re-checked row read as a disabled imperative, so the row now says 're-checked · unsettled' before the chip that performs the re-check; the card-level fresh-run block has its two goldens; the carved-out class follows the view-topology rule (a container ends with Container)."
- 2026-10-17T17:16:35Z @neo-fable cross-referenced by #722
- 2026-10-17T18:28:30Z @neo-gpt-emmy cross-referenced by #505
- 2026-10-17T18:29:12Z @neo-fable-clio cross-referenced by #724
- 2026-10-17T18:36:35Z @neo-fable-clio cross-referenced by #726
- 2026-10-17T18:43:00Z @neo-fable cross-referenced by #727
- 2026-10-17T18:48:15Z @neo-fable referenced in commit `be34cb5` - "chore(agentos): merge dev after #723 into the fresh-run branch (#475)"
- 2026-10-17T19:10:52Z @tobiu referenced in commit `fa1a460` - "feat(agentos): the setup card offers a fresh run where a re-check could not settle an effect (#475) (#721)

* test(harness): the vessel's broker settles an interrupted host-file effect by its own observation and the run goes on (#475)

Over the pinned Brain's own modules: a write-secrets whose acknowledgement write was rejected leaves a pending receipt; once the host shows the secret files, the next request settles it by observation with no plane of the target answering, the handler stays at one run, and the following effect writes the carrier.

* feat(harness): the setup broker starts a fresh run on request and names the run it retires (#475)

setupEvaluate({target, fresh: true}) binds a new record for the target, the shell's form of a CLI run that names no run id, and answers {ok, evaluation, retiredRunId}. The record the broker held is neither read nor written, so a record that cannot be read does not stand in the way; the new record is bound once it is on disk, so a rejected write leaves the binding where it was. A fresh that is not a boolean is refused by name.

The ShellPlane addon forwards the flag as given; main is the one validator.

Arms over the pinned Brain's own modules: the retired record stays byte-identical and takes no later consent, no effect is replayed, a later boot resumes the fresh run, an unreadable bound record is left as it is, a rejected record write binds nothing.

* feat(agentos): the setup card offers a fresh run where a re-check could not settle an effect (#475)

A host-file effect whose receipt was interrupted and whose result the host no longer shows stays reconcile-required through every re-check, and the card had re-check alone. The operator's own re-check in this card now earns the row a quiet second chip, start a fresh run; the first press says what it leaves behind, the second sends setupEvaluate({fresh: true}), and the broker binds a new record for the same target while the retired one stays on disk as history. A read that refuses with no live row left offers the same exit beside its consent line. The preset choices under Other choices move into their own PresetCards class.

* fix(agentos): a re-checked row says the fact before its controls, the card-level exit gets its goldens, the preset cards end with Container (#475)

The design seat's read of the built rows: 're-check' on the re-checked row read as a disabled imperative, so the row now says 're-checked · unsettled' before the chip that performs the re-check; the card-level fresh-run block has its two goldens; the carved-out class follows the view-topology rule (a container ends with Container)."
- 2026-10-17T19:10:53Z @tobiu closed this issue

