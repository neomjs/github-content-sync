---
id: 956
title: 'The placement probe tells unobserved from unsupported, with a next step'
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-fable
createdAt: '2026-10-09T21:28:35Z'
updatedAt: '2026-10-09T23:52:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/956'
author: neo-fable-clio
commentsCount: 2
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
closedAt: '2026-10-09T23:52:23Z'
---
# The placement probe tells unobserved from unsupported, with a next step

## Context

Graduated from [D#19493](https://github.com/orgs/neomjs/discussions/19493) (§4 O1, adopted for v1 by the walkers' convergence on 2026-10-09; the probe's contract from Sophie's §5.2 sweep, point 4). The operator's installed read of the Create door on Candidate F (D#19493, comment 18836049): the host budget could not be read, every preset was *refused*, the ledger row read `failed · no supported preset fits this host` with `re-read` as the only exit, and nothing named the cause or the next step. A stranger's first run ended there.

## The Problem

A failed read and a measured shortfall produce the same screen. The card renders what the probe says (`SetupAsks.recommendationText`, the preset verdicts), and the probe today says `the host budget is incomplete (unobserved: vmInfo, containerStats, …)` and `host pressure is unknown: a local preset is not called a fit` (`ai/services/fleet/probePlacement.mjs:231`, `:246`) — internal reader names, no cause a reader can act on, and no distinction between *we could not look* and *we looked and it does not fit*. The card cannot repair this alone: a card leaf (Institution #351's first card leaf, Mnemosyne's) renders a cause and a next step only when the probe gives one.

## The Architectural Reality

- `probePlacement` (`ai/services/fleet/probePlacement.mjs:67`) already records `observed` per reader and an `uncertainty` list with one `{reader, reason}` per failed read (`:70–90`, `:151` for the VM reservation policy); the budget it feeds is marked incomplete when a read fails.
- `fitsPreset` (same file) refuses with distinct reasons: an unobserved host, an observed memory shortfall, observed swapping — Sophie ran it against four fixtures on `daff56b2` (D#19493, 18842602): observed fit succeeds; the three refusals carry different reasons. The distinction exists in the data and is lost in the words.
- The recipe's placement step (`ai/services/fleet/firstRunRecipe.mjs:256–267`) turns `recommendPlacement` into `recommended: …` or `nothing recommended; possible: …`; the card's row and the window title read that status.
- The VM reader is the only reader that can say "no VM observed"; whether Docker Desktop is stopped is its word, never an inference from an unspecified failed read.

## The Fix

The probe's verdict carries two classes and two new fields, in the probe and in the recipe's status:

1. **Unobserved** (a reader failed or is absent): the verdict is an *unverified* recommendation — the preset table is applied to what was observed, the result is marked `unverified` with the list of readers that did not answer, and `reason` + `nextStep` name the cause and the one thing to do (the VM reader absent → "Docker Desktop is not running · start it, then re-read"; a reader that threw → its message in the reader's own words). The recipe's status is not `failed`; it is `ok` with an `unverified` recommendation, so the door continues.
2. **Observed unsupported** (a measured shortfall or swapping): the verdict stays a refusal with its measured reason, as today, and `nextStep` names what would change it (free memory, close the swapping consumer, choose the hosted preset). A measured shortfall never becomes a fit.
3. One vocabulary contract for `reason` and `nextStep` (short, reader-facing, one sentence each; the card may render them verbatim), documented once, so the card never re-words the probe.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `probePlacement` result: `uncertainty[]` entries | `probePlacement.mjs:70–90` | each entry gains `cause` (reader-facing) and `nextStep`; the VM reader's "daemon absent" is the only source of a Docker sentence | a reader with no cause → `cause: 'did not answer'`, `nextStep: 're-read'` | the probe's JSDoc | unit arms per reader class |
| `fitsPreset` verdicts | same file | a refusal carries `kind: 'observed'` with its measured reason; an unobserved budget yields `kind: 'unverified'` with the preset's fit on the observed facts and the missing readers | both kinds keep today's reason strings for the ledger's title | JSDoc | Sophie's four fixtures become the arm: fit, unobserved host, memory shortfall, swapping |
| `firstRunRecipe` placement step status | `firstRunRecipe.mjs:256–267` | `ok` with `unverified` recommendations when nothing is refused on observed facts; `failed` only when an observed refusal leaves no preset | unchanged status words elsewhere | the recipe's step table | unit arm: unobserved host → `ok · unverified`, observed shortfall on every preset → `failed` with reasons |
| The reader-facing vocabulary | this leaf | one sentence each for `cause` and `nextStep`; the card renders them verbatim (Institution #351's card leaf consumes) | — | one table in the probe's JSDoc | the Institution leaf's unit arm on the words |

Decision Record impact: none (the probe's data already carries the distinction; this is its contract and words).

## Acceptance Criteria

- [ ] AC-1 Red first: with the VM reader absent and the rest observed, the probe answers an `unverified` recommendation with `cause` naming the VM reader and `nextStep` "start Docker Desktop, then re-read"; today it refuses every preset.
- [ ] AC-2 Sophie's four fixtures: observed fit → fit; unobserved host → `unverified` with the missing readers; observed memory shortfall and observed swapping → `observed` refusals with their measured reasons and a `nextStep`; a measured shortfall is never a fit.
- [ ] AC-3 The recipe's placement step reads `ok · unverified` for AC-1's case and `failed` only when observed refusals leave no preset; the window title's progress line follows.
- [ ] AC-4 The vocabulary table exists in the probe's JSDoc; every `cause`/`nextStep` the arms produce is in it.
- [ ] AC-5 (post-merge, with Institution #351's card leaf) The installed walk's Docker-stopped arm (#534) shows the cause and the next step on the card, and the door continues.

## Out of Scope

The card's rendering (Institution #351's first card leaf); durations per step (its third card leaf); diagnosing anything the readers do not report.

## Avoided Traps

- Turning "never blocks" into "always fits": the unobserved class recommends unverified, the observed class still refuses.
- Guessing Docker from a generic failed read: only the VM reader's own word says it.
- A second vocabulary in the card: the probe's words are rendered, not re-worded.

## Related

D#19493 §4 O1 and OQ-2; Institution #351 (row 1) and #534 (its walk); #858 (the recipe's forge-connection row); the operator's installed read (D#19493, 18836049); Sophie's fixtures (18842602).

Live latest-open sweep: the latest 20 open issues, created-descending, at 2026-10-09 21:2xZ; no equivalent. A2A in-flight claim sweep: none. Memory Core rationale: D#19493 is the record.

Origin Session ID: cf93d406-6f17-4f10-9f72-9768482edfb1

Retrieval Hint: "placement probe unverified recommendation cause nextStep observed refusal Docker VM reader"

## Timeline

- 2026-10-09T21:28:36Z @neo-fable-clio added the `enhancement` label
- 2026-10-09T21:28:37Z @neo-fable-clio added the `ai` label
- 2026-10-09T21:28:37Z @neo-fable-clio added the `agent-os` label
- 2026-10-09T21:33:39Z @neo-fable-clio added parent issue #351
- 2026-10-09T21:36:28Z @neo-fable-clio cross-referenced by #858
- 2026-10-09T21:38:32Z @neo-fable assigned to @neo-fable
### @neo-fable - 2026-10-09T21:38:34Z

**Intake (row 1's builder, 21:4xZ) — taken.** Prescription check against `probePlacement.mjs` and `firstRunRecipe.mjs` at dev `0b678f47`: the data already splits the classes (`observed` per reader, `uncertainty[] = {reader, reason}`, `fitsPreset`'s distinct reason strings), and `evaluatePlacement` collapses them into `ok`/`failed` — the ticket's reading holds. One refinement I carry into the leaf: `recommendPlacement` must propagate `kind` per preset so the recipe's `ok · unverified` is computed from the verdicts, not re-derived from the budget. Arms: AC-1 red first; the four fixture classes; the recipe's two statuses; the vocabulary table as the single source the Institution card leaf renders. Reviewer at PR time: a GPT seat (Sophie ran the fixtures). Branch `fable/956-probe-cause-next-step`.

🪢 Mnemosyne (Claude Fable 5.1 · Claude Code) · session 882aedde-7578-4231-9183-8160f948db6a

- 2026-10-09T21:52:45Z @neo-fable cross-referenced by PR #959
- 2026-10-09T21:56:27Z @neo-fable referenced in commit `3617f99` - "docs(fleet): the vocabulary's docblock names the card, not a ticket (#956)"
- 2026-10-09T23:13:27Z @neo-fable referenced in commit `4d8a0e9` - "fix(fleet): a negative balance and observed swap stay observed refusals on the unknown-data path (#956)"
- 2026-10-09T23:14:05Z @neo-fable referenced in commit `97cf0c9` - "test(fleet): the negative-evidence arm's comments describe the hosts, not the review (#956)"
- 2026-10-09T23:23:43Z @neo-fable referenced in commit `ceb4e08` - "test(fleet): the negative-evidence hosts are pinned through the recipe's placement status (#956)"
### @neo-fable - 2026-10-09T23:31:06Z

**Handover (session sunset, 2026-10-10 00:4xZ) — the lane stays mine; PR #959 is in round two.** State: branch `fable/956-probe-cause-next-step` at `ceb4e08a` (pushed; the clone is back on `dev`); Sophie's R1 CHANGES_REQUESTED repaired (a signed balance is guarded by `Number.isFinite`, swap in use reads swapping before the total is needed), her four hosts pinned at the verdict and through the recipe's placement status; 44/44 fleet arms + the probed recipe arm; author responses on the PR; her six-specimen falsifier passes; the R2 verdict is pending. One reading flagged to her: a swapping host with its total unread refuses the local presets and leaves hosted `unverified`, so the step reads `ok · unverified: hosted`, per AC-3's own words. Pickup for my next session: read the PR thread's latest review; if approved, the merge is the operator's; if changes are requested, address them on the branch. Nothing else is owed on this ticket; AC-5 is post-merge with Institution #351's first card leaf.

🪢 Mnemosyne (Claude Fable 5.1 · Claude Code) · session 882aedde-7578-4231-9183-8160f948db6a

- 2026-10-09T23:52:23Z @tobiu referenced in commit `9a73e15` - "feat(fleet): the placement probe tells unobserved from unsupported, with a cause and a next step (#956) (#959)

* feat(fleet): the placement probe tells unobserved from unsupported, with a cause and a next step (#956)

Every uncertainty entry carries the reader's own word plus a cause and a next step from one
vocabulary; a preset verdict carries a kind — observed (a fit, or a refusal with its measured
reason) or unverified (a read did not answer; the preset's fit on the observed consumers alone,
never an affirmative fit) — with the same two sentences; the recipe's placement step reads
ok · unverified instead of failed when the host did not answer, and failed only when observed
refusals leave no preset. The Docker sentences come only from the Docker readers' own error text.

* docs(fleet): the vocabulary's docblock names the card, not a ticket (#956)

* fix(fleet): a negative balance and observed swap stay observed refusals on the unknown-data path (#956)

* test(fleet): the negative-evidence arm's comments describe the hosts, not the review (#956)

* test(fleet): the negative-evidence hosts are pinned through the recipe's placement status (#956)"
- 2026-10-09T23:52:23Z @tobiu closed this issue

