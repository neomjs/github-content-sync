---
id: 724
title: 'Add an agent: the empty-memory row, the focus ring and the lock glyph'
state: OPEN
labels:
  - bug
  - agent-os
  - ai
  - design
assignees:
  - neo-gemini-pro
createdAt: '2026-10-17T18:29:11Z'
updatedAt: '2026-10-17T21:25:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/724'
author: neo-fable-clio
commentsCount: 0
parentIssue: 505
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
# Add an agent: the empty-memory row, the focus ring and the lock glyph

## Context

Operator-observed 2026-10-17 ~20:10 Berlin, adding `neo-gemini-maryam` on the installed Fleet Manager (organism staged 2026-10-14T21:26Z: Institution afabdef6, Brain e219267e). Eleven memory candidates were offered; the row *Start with empty memory* sat last, at the bottom of the list, unselected; the submit refused with "Choose the agent whose memory this seat continues, or Start with empty memory."; the chosen row showed the stock focus ring over the selection tint; the one primary action read 🔒 Add agent. His words: "the amount of friction is intense". The submit then met a plane refusal (`defineAgent` without an admission owner) — that is #726's and the recipe's matter, not this ticket's. Emmy's installed reproduction (#505, comment 6238570247) confirmed the walk and added three observations, folded in below.

## The Problem

A new agent is this form's common case, and its right answer — no memory to continue — is the one choice the form hides: last in a group that grows with every seat and checkout the host has ever had (eleven rows here), with no preselection once more than one candidate exists. The operator scrolls, misses it, submits, reads a validation sentence, scrolls again. Smaller things compound it: the selected row carries the engine's stock focus ring on top of the form's own selection tint (a double frame, the only hard-blue element on the surface); the primary button's lock glyph reads, in this app's own language, as "locked" rather than "a token will be asked"; the copy note stays under a choice it no longer describes; and the token sheet asks without saying for whom.

## The Architectural Reality

- `apps/agentos/util/AddAgentFlow.mjs:292` `memoryChoices`: candidates first, the empty row last, by design ("then *Start with empty memory* as the last row of the same group"); `apps/agentos/view/fleet/instances/AddAgentForm.mjs:37`: "one candidate is preselected, several wait for a choice" — `AddAgentFlow.preselectedMemory` (`AddAgentForm.mjs:699`) selects nothing when several candidates exist; `AddAgentFlow.mjs:351` the validation sentence; `AddAgentFlow.mjs:78` `MEMORY_COPY_NOTE`.
- `apps/agentos/view/fleet/instances/MemoryCandidateList.mjs` is a `neo-list`; `resources/scss/src/apps/agentos/fleet/instances/MemoryCandidateList.scss` styles the rows and `.neo-selected` (signal tint + weight) but sets no focus rule, so the engine's `list/Base.scss:67` `--list-item-focus-outline` paints the stock ring over the selection; the Observatory sets its own quiet ring at `resources/scss/src/apps/agentos/fleet/goldenpath/ObservatoryContainer.scss:251` — the precedent.
- `AddAgentForm.mjs:338` `iconCls: 'fa fa-lock'` on the submit (`fm-add-submit`, the signal-tinted primary per the button hierarchy, ledger D1); the help line `On Add, the app asks once for this agent's token.` (`AddAgentForm.mjs:406`) already carries that fact in words.
- The candidates come from the Fleet (`fleetMemoryCandidates`, a read-observe verb); their order and count are the host's — the form cannot assume few.
- The native token sheet (the one-token journey, #515) says "GitHub or GitLab" and names neither the agent nor the plane the token binds to (Emmy's reproduction).

## The Fix (design read; the Institution steward shapes the code)

1. **The empty row first and preselected** whenever the form opens for a username that is none of the candidates' agents — the new-agent case; the candidates follow under their lead ("Continue one of these agents' memory?") as the alternative. A single candidate that matches stays preselected as today. The validation sentence then fires only when the operator deselects everything.
2. **Eleven rows are a list, not a frame:** cap the candidate group's height (≈ 5 rows) with its own scroll and the setup door's foot fade (#644's idiom), so the choice and the fields after it stay on one screen.
3. **One frame per row:** a `:focus-visible` rule for `.fm-memory-candidates .neo-list-item` — an inset 1px ring in `--fm-signal` at a low mix, and no outline when the row is `.neo-selected` and focused (the tint already says it). The same rule family as `ObservatoryContainer.scss:251`.
4. **No lock on the primary:** drop the glyph (the help line says the token is asked once), or a key glyph if a glyph must stay; the lock stays reserved for a disabled or gated state, which this is not.
5. **The copy note belongs to the candidates:** `Its notes are copied, never moved; the original stays where it is.` shows under a chosen candidate and hides for the empty row (it stayed after an explicit empty-memory click).
6. **The token sheet names what it asks for:** the agent (`neo-gemini-maryam`), the plane the token binds to, and the forge the form chose (GitHub) — not "GitHub or GitLab".

## Contract Ledger

No wire surface changes: the `memoryImport` values, the `agentDefinitionAccepted` event and the Fleet reads are untouched. Decision Record impact: none (design conformance under #505's button hierarchy and focus idiom).

## Acceptance Criteria

- [ ] AC-1 (component arm, `test/playwright/component/apps/agentos/AddAgentForm.spec.mjs`): with ≥ 2 candidates and a username matching none, the empty row is the first row and selected on open, and submit passes without the validation sentence; with exactly one candidate whose agent matches the username, that candidate is preselected (today's behavior kept).
- [ ] AC-2 (visual, the cockpit visual suite): the memory frame with eleven candidates fits the card at the suite's viewport with its own scroll; goldens dark + light for the selected + focused row — one frame, no stock outline.
- [ ] AC-3 (visual): the submit reads `Add agent` with no lock glyph (or a key glyph, decided in the PR with the capture shown); `[disabled]` keeps its own look.
- [ ] AC-4 (post-merge, installed): the operator adds a new agent on the next Institution #12 candidate without scrolling for the memory choice — receipt on the Row 1 walk (2026-10-23).
- [ ] AC-5 (component arm + capture): the copy note is absent while the empty row is chosen and present under a chosen candidate; the token sheet's words carry the agent and the forge (asserted where the sheet is testable, else the capture in the PR).

## Out of Scope

The plane refusal after the token prompt — #726 (the Add form reads admission before the token; the plane-state line) and, for the Connect door, #351's deferred amendment; the candidate discovery itself (what the Fleet lists and why); *New agent* reopening the retained failed draft rather than resetting it (source-intended per Emmy's read — a design question for #505's board, not this leaf).

## Avoided Traps

Guessing a candidate from the username (memory is offered, never guessed — the form's own rule stays); hiding the candidates behind a disclosure (the operator must still see that memory exists before adding); a global `neo-list` focus change in the engine theme (the Observatory precedent is local, this stays local).

## Related

#505 (parent, the design epic), #726 (the admission read + the plane-state line), #644 (the setup door's foot fade — the scroll idiom), #507 (the default perspective), #12 (the walk), #475 / #721 (the setup card — the lock-free primary idiom there), #515 (the one-token journey), Brain #805 (the admission guard the plane refused with), #505 comment 6238570247 (Emmy's installed reproduction).

Live latest-open sweep: checked the latest 20 open Institution issues at 2026-10-17T18:24Z (newest #722); nearest #687 (target positions for harness windows) and #644 (the setup door's foot fade) — neither on the Add form; exact search `Add agent form` (open, org-wide) returned only the epics #7 and #13. A2A in-flight claim sweep (last hour): none on the Add form. MC sweep ("Add an agent form existing memory candidates list Start with empty memory last row preselected lock icon"): initialization noise only, no prior decision. Own-assignment sweep: #505 (this leaf's parent), #507, #351 — no overlap. Structure map: N/A (an Institution app surface; the Brain-hosted map does not cover it).

unowned-rationale: filed by the design seat from the operator's walk; the Institution steward self-selects; the design read on the PR's captures stays mine.

Origin Session ID: cd7bf4ee-8722-43be-baf8-29f9ed658722

Retrieval Hint: "Add an agent form Start with empty memory last row focus ring lock glyph eleven memory candidates"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session cd7bf4ee-8722-43be-baf8-29f9ed658722


## Timeline

- 2026-10-17T18:29:13Z @neo-fable-clio added the `bug` label
- 2026-10-17T18:29:13Z @neo-fable-clio added the `agent-os` label
- 2026-10-17T18:29:13Z @neo-fable-clio added the `ai` label
- 2026-10-17T18:29:13Z @neo-fable-clio added the `design` label
- 2026-10-17T18:30:10Z @neo-fable-clio added parent issue #505
- 2026-10-17T18:36:35Z @neo-fable-clio cross-referenced by #726
- 2026-10-17T18:43:00Z @neo-fable cross-referenced by #727
- 2026-10-17T20:22:55Z @neo-gemini-pro cross-referenced by PR #729
- 2026-10-17T21:25:20Z @neo-gemini-pro assigned to @neo-gemini-pro

