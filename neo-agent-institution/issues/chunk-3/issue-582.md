---
id: 582
title: System reviews and consents to this installation's seat move
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-06T13:58:16Z'
updatedAt: '2026-10-06T16:37:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/582'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 19428 ADR 0034 §2.3 item 11: moving the seat root is a named broker'
  - '[x] 573 An installed shell moves its seats to the default seat root'
blocking: []
closedAt: '2026-10-06T16:37:08Z'
---
# System reviews and consents to this installation's seat move

## Context

The occupied-root transition in #573 is the placement step of neomjs/neo-agent-brain#571. Its accepted [planner disposition](https://github.com/neomjs/neo-agent-institution/issues/573#issuecomment-6017036252) separates the global placement transition from each peer's later adoption. This leaf is the System consent surface explicitly excluded from that ticket.

## The Problem

The operator has no supported UI to inspect the installation's proposed move, consent to that exact plan, or read its outcome after relaunch. The installed root is adopted app data; changing the root alone would strand the registry bindings. The transition belongs to the shell, and a held Fleet boot must not hide its explanation.

## The Architectural Reality

At `ada/573-seat-root-move@f62ec0c2`, `harness/seatRootMove.mjs` owns `planSeatMove`, `consentSeatMove`, `readSeatRootMove` and `settleSeatRootMove`. Main retains `seatMoveOutcome`, but has no seat-root IPC handlers. On current dev, `harness/preload.cjs` and Institution's `src/main/addon/ShellPlane.mjs` expose plane/setup brokers. `apps/agentos/view/system/Container.mjs` projects plane diagnostics and declares them observe-only.

Design authority: #573, Out of Scope, names “The System view's consent surface (old and new roots, the dry-run rows, consent, relaunch, the outcome)” as this sibling. ADR 0034 §2.3 requires named capabilities and sender validation; Grace owns item 11's declaration in neomjs/neo#19428 / neomjs/neo#19429 before this implementation becomes merge-eligible.

## The Fix

Add a separate **Seat folders on this machine** section to System. Its controller reads the shell directly through ShellPlane, independent of the bound plane or Fleet health. The section shows the recorded root and latest transition outcome, offers a dry-run review of both roots and every row, then an explicit **Move seats and relaunch** consent. Explain that homes are copied and checked, that space and time are required, and that source folders are retained in an archive.

Use a Store/Model and List for the plan, a controller for reads/intents and local view state for rendering. Add the narrow main broker beside `setupBroker.mjs`, wire main/preload/ShellPlane, and call the existing transition helpers. Roots and archive paths are main-owned; only a fingerprint crosses with consent. Serialize consent in the broker so two windows cannot record crossed moves. This does not duplicate the mover.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| New `seatRootStatus()` / `shell-seat-root-status` | #573; ADR 0034 item 11 companion | Read `{packaged, root, pending, outcome}`; each record is its value, null if absent, or `{unreadable: reason}`; status works with Fleet boot held | Unknown/unreadable status preserves other readable fields and boot outcome, displays reason and disables consent; no-shell is named | JSDoc, native-vessel guide | Broker + rendered held-boot controls |
| New `seatRootPlan()` / `shell-seat-root-plan` | `planSeatMove` at f62ec0c2 | `{state,code?,reason?,from,to,rows,fingerprint}`; every row shown with id, source, destination, disposition and reason | Refusal or failed read invalidates the previous consentable plan | JSDoc | Plan/refusal and stale-response tests |
| New `seatRootConsent({fingerprint})` / `shell-seat-root-consent` | `consentSeatMove` at f62ec0c2 | Re-plan in main, persist matching inputs, then relaunch on `consented` | Producer refusals carry `{state: 'refused', code, reason}`; malformed requests and read/write failures reject the invoke; none relaunch or retry | JSDoc | Real temporary records + concurrent consent and changed-plan controls |
| System section and ShellPlane transport | Existing System, setup broker patterns | Installation-scoped UI; only named capabilities; every handler checks sender before reads/effects | Browser capability unavailable is explicit; no Fleet request fallback | View JSDoc, guide | Unit, component/visual and broker wiring evidence |

## Acceptance Criteria

- [ ] All three channels are declared by the Engine ADR companion, forwarded by preload/ShellPlane, sender-validated and registered even if the Fleet child cannot boot.
- [ ] System distinguishes the local installation's seat folders from the connected plane's diagnostics. It shows old/new roots and every dry-run row before consent, including untouched/refused rows and reasons.
- [ ] Consent sends only the reviewed fingerprint. Busy, unreadable, refused, stale or superseded plans cannot consent; failed invokes do not display success or silently retry.
- [ ] Main invokes the existing transition helpers with its own roots. Concurrent requests are serialized; a changed plan writes nothing; one accepted request records once and relaunches once.
- [ ] After relaunch the surface reports none, refused, held, committed and retirement-held honestly. A committed placement does not claim a peer's session/memory adoption.
- [ ] Focused unit and rendered tests cover happy review/consent, browser/no-shell, failed reads, changed plan, held boot, escaped paths/reasons and all rows at narrow/wide widths. Validation names the exact paired source revisions.

## Post-Merge Validation

Candidate C's installed walk uses this surface to consent and reads the outcome. Sophie's relocated session/memory witness and each peer's adoption remain neomjs/neo-agent-brain#571 and #12.

Residual-Owner: neomjs/neo-agent-brain#571

## Out of Scope

The copy/verify/rebind algorithm and transition recovery (#573, neomjs/neo-agent-brain#900); arbitrary destination selection; source archive deletion; peer memory-import choice; actual installation or seat moves as part of source tests.

## Avoided Traps

A Fleet verb cannot explain a boot it holds. A raw path from the renderer cannot choose installation placement. A root-record setter cannot replace the coordinated transition. The plane diagnostics remain observe-only; local placement controls have their own scope.

## Decision Record impact

Depends on neomjs/neo#19428 (PR neomjs/neo#19429), Grace's amendment of Engine ADR 0034 §2.3 item 11; aligned with its §2.3.4 sender boundary. No additional authority record.

## Related and retrieval

Blocked by #573. Parent outcome: neomjs/neo-agent-brain#571. Integration: #12.

Sweeps at 2026-10-06 13:58Z: latest 20 open issues (#573 through #312), no duplicate; 30 recent all-state A2A messages, overlapping Grace offer resolved into Engine ADR / Emmy Institution split, confirmed by Ada. MC problem queries retrieved the installed-root and operator-memory context; historical empty-destination inference was superseded by live source. Own assignments: #42 body read, view-debt measurement only. Live organization search “seat root consent” returned no ticket. Brain structure map ran this session; owner here is Institution `harness/` broker and `apps/agentos/view/system/`, using setup siblings, no new subsystem.

Origin Session ID: d0d0bed3-7ce4-4bce-a16d-59589484aec0

Retrieval Hint: “occupied seat root System consent fingerprint held Fleet boot Gap 13”

🪡 Emmy · @neo-gpt-emmy

Prescription checked: `harness/setupBroker.mjs` and `apps/agentos/view/system/Container.mjs` — the separate installation broker owns host reads/consent; the separately labelled System section projects it. Independent source read confirmed this placement and rejected folding placement into setup's separate recipe record. Ada confirmed ownership in the 14:01Z handoff. Producer `3487e02` adds typed consent refusal codes; the consumer uses state/code, never parses reason text. Pre-brief: the newly filed node was not yet indexed; live ticket/source and the failure-mode memory sweep supplied the brief.


## Timeline

- 2026-10-06T13:58:16Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-06T13:58:45Z @neo-gpt-emmy marked this issue as being blocked by #573
- 2026-10-06T13:58:49Z @neo-gpt-emmy added the `enhancement` label
- 2026-10-06T13:58:49Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-06T13:58:50Z @neo-gpt-emmy added the `ai` label
- 2026-10-06T14:03:20Z @neo-opus-grace cross-referenced by PR #19429
- 2026-10-06T14:08:20Z @neo-opus-grace referenced in commit `dd37a33` - "docs(adr): item 11 answers with the transition's own DTOs, and an unreadable record never hides the boot outcome (#19428)

Per the consumer read for neomjs/neo-agent-institution#582: the channels answer
the producer's DTOs unwrapped (#573 at 3487e02). A refusal is
{state: 'refused', code, reason}, deciding on the code and showing the
sentence; the shell's own refusals use the same shape. Status reports a
record it cannot read as {unreadable}, keeps the boot outcome, and adds
packaged. Custody names the consent channel as the inputs' writer and the
boot's deletion of refused inputs."
- 2026-10-06T14:11:26Z @neo-gpt-emmy cross-referenced by #12
- 2026-10-06T14:17:20Z @neo-gpt-emmy added parent issue #571
- 2026-10-06T14:21:42Z @neo-gpt-emmy marked this issue as being blocked by #19428
- 2026-10-06T14:25:47Z @neo-opus-grace cross-referenced by #19428
- 2026-10-06T14:26:56Z @neo-gpt cross-referenced by #477
- 2026-10-06T14:47:49Z @neo-opus-ada referenced in commit `cd0c01f` - "build(harness): the packaged app carries the seat move module (#573)

main.mjs imports seatRootMove.mjs, so electron-builder's files allowlist and pack.spec's expected main-module closure name it (Emmy's #582 packaging finding)."
- 2026-10-06T14:58:08Z @tobiu referenced in commit `cec2fcc` - "docs(adr): ADR 0034 §2.3 item 11 declares the seat-root move as a named broker (#19428) (#19429)

* docs(adr): ADR 0034 §2.3 item 11 declares the seat-root move as a named broker (#19428)

The FM switch moves every seat from the installed shell's app-data root to the
default seat root, and the operator consents through the System view
(neomjs/neo-agent-institution#573, #582). Item 11 declares the three channels
that view needs, in items 8 and 10's form: shell-seat-root-status
(pull-shaped, answered while the Fleet boot is held), shell-seat-root-plan
(the Brain's dry run with its fingerprint) and shell-seat-root-consent (an
unchanged plan only; persists the move's inputs and relaunches, or refuses by
code). Custody: main is the only writer of the move's inputs; no credential
crosses.

* docs(adr): item 11 answers with the transition's own DTOs, and an unreadable record never hides the boot outcome (#19428)

Per the consumer read for neomjs/neo-agent-institution#582: the channels answer
the producer's DTOs unwrapped (#573 at 3487e02). A refusal is
{state: 'refused', code, reason}, deciding on the code and showing the
sentence; the shell's own refusals use the same shape. Status reports a
record it cannot read as {unreadable}, keeps the boot outcome, and adds
packaged. Custody names the consent channel as the inputs' writer and the
boot's deletion of refused inputs.

* docs(adr): item 11 codes a consent refusal and the shell's own, and the plan answers the seats' steady state (#19428)

The Brain's dry run refuses with a reason alone, so only a consent refusal and the shell's refusals carry a code. A plan for seats already at the target root refuses nothing-to-move before the Brain runs, since moveSeatHomes throws on equal roots. A committed move keeps its inputs, which status still reports as pending."
- 2026-10-06T15:10:49Z @neo-opus-ada cross-referenced by PR #584
- 2026-10-06T15:13:38Z @neo-opus-ada cross-referenced by #571
- 2026-10-06T15:19:16Z @neo-gpt-emmy referenced in commit `eed4144` - "feat(agentos): review and consent to the installation seat move (#582)"
- 2026-10-06T15:19:19Z @neo-gpt-emmy cross-referenced by PR #585
- 2026-10-06T15:52:45Z @tobiu referenced in commit `9a025ab` - "feat(harness): an installed shell moves its seats to the default seat root, with the Brain pin at a8dd1ae4 (#573) (#584)

* feat(harness): a consented move of the seats settles at boot, before the first-launch choice (#573)

seatRootMove.mjs holds the installation-wide move. Consent persists only the
inputs of the plan the operator saw (the Brain's moveSeatHomes dry run). At the
next boot, before settleSeatRoot and before any Brain child, the shell:
- rules out a live registry writer;
- re-plans, then moves through the Brain's one-shot;
- reads the registry back, then commits by writing the root record (moved);
- retires the old folders into a dot-archive under the old root.

A move that cannot commit brings the old bindings back and spends the consent.
After the commit, only the retirement resumes. A move that can neither go on
nor come back holds the boot. The live arm runs the real one-shot against a
Brain root.

* feat(harness): the consented move carries its id to the Brain, which owns only stages marked with it (#573)

Consent records a moveId, and the boot's plan and move steps pass it to the
Brain's moveSeatHomes (NEO_HARNESS_SEAT_MOVE_ID). A staging folder is that
move's own only when it carries the id, so an interrupted copy is discarded and
redone, while any other occupant stops the move (neomjs/neo-agent-brain#901,
review round 1, RA-6).

* feat(harness): a refused consent names its code beside its sentence (#573)

consentSeatMove refuses with {state: 'refused', code, reason}: no-seat-root, already-consented, plan-refused, plan-changed or nothing-to-move, so the consent surface decides on the code (ADR 0034 §2.3's named-refusal rule, item 11 in Grace's amendment) and keeps the sentence for the person and the log.

* build(harness): the packaged app carries the seat move module (#573)

main.mjs imports seatRootMove.mjs, so electron-builder's files allowlist and pack.spec's expected main-module closure name it (Emmy's #582 packaging finding).

* style(harness): the pack spec's blocks align, as the preflight repaired them (#573)

* build(deps): the declared Brain pin moves to a8dd1ae4, where a moved seat starts at its new root (#573)

* fix(harness): the move gives back only what it provably did, its archive is a folder in the old root, and a held retirement holds the boot (#573)"
- 2026-10-06T15:58:09Z @neo-gpt-emmy referenced in commit `155f5c9` - "feat(agentos): review and consent to the installation seat move (#582)"
- 2026-10-06T15:58:10Z @neo-gpt-emmy referenced in commit `7ebce5a` - "fix(agentos): reflect settled seat-move and Fleet hold outcomes (#582)"
- 2026-10-06T16:37:08Z @tobiu referenced in commit `df65934` - "feat(agentos): review and consent to the installation seat move (#582) (#585)

* feat(agentos): review and consent to the installation seat move (#582)

* fix(agentos): reflect settled seat-move and Fleet hold outcomes (#582)"
- 2026-10-06T16:37:08Z @tobiu closed this issue
- 2026-10-06T17:58:36Z @neo-gpt cross-referenced by #589

