---
id: 341
title: 'Home offers Connect a plane on first run, and doors for the team'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-vega
createdAt: '2026-09-30T09:14:10Z'
updatedAt: '2026-09-30T14:56:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/341'
author: neo-opus-vega
commentsCount: 1
parentIssue: 9
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-30T14:56:27Z'
milestone: FM v1
---
# Home offers Connect a plane on first run, and doors for the team

## Context

The operator said on 2026-09-26 that Home "looks boring", and on 2026-09-28 asked that before any visual work Home define what it enables for the returning team and for a first-time operator, and that the lede's visible font mismatch be fixed (relayed on #244). My definition on #244 answers that ([comment](https://github.com/neomjs/neo-agent-institution/issues/244#issuecomment-5900445503)). Clio's #12 AC-4 sign-off names *Connect a plane* as the plane-less shell's one action on Home, and Grace decided the headline question for returning readers on #244.

This ticket takes the part of that definition that needs no roster read. #244 keeps what does: the team line, and the canvas.

## The Problem

Home is one HTML string: an eyebrow, a headline, and a lede that ends in a prose direction ("Select Fleet in the rail"). It has no action and no live fact. A first-time operator in a packaged shell without a plane cannot start from Home, and the returning team sees neither the plane's state nor a way into the other views.

The lede declares no `font-family`, so it inherits the theme's body face ("Source Sans 3") under a display line set in the system-ui stack. Measured on Institution `dev@1d592e1`.

## The Architectural Reality

- `apps/agentos/view/Viewport.mjs`: the Home tab item is `{ntype: 'component', cls: ['agent-welcome'], html: …}`. Its styles live in `resources/scss/src/apps/agentos/Viewport.scss` (`.agent-welcome*`).
- `ViewportController#mountPlaneSetup` reads `readPlaneStatus()` at boot and mounts `PlaneSetupPanel` for a packaged shell without a plane. `onAttachPlane()` → `showPlaneSetup()` brings the card back on request.
- `instanceState` on the Viewport provider is the chrome switcher's connection verdict (`ok` · `limited` · `off` · `starting`). The cockpit's formula writes it, and `SwitcherButton`'s `INSTANCE_STATE_WORDS` is its vocabulary.
- Keeper views are `container.Base` classes at `apps/agentos/view/<view>/Container.mjs`, with their styles at `resources/scss/src/apps/agentos/<view>/Container.scss`. System is the sibling.

## The Fix

- `AgentOS.view.home.Container` replaces the HTML string. It binds `instanceState` and a new Viewport data key, `shellPlaneConfigured`.
- `mountPlaneSetup` publishes `shellPlaneConfigured` from the status it already reads:
  - `false` for a packaged shell without a plane;
  - `true` for a packaged shell with one;
  - `null` for a browser build or a failed read.
- A first-time reader (`shellPlaneConfigured === false`) gets the headline, the lede and one action, *Connect a plane*, which calls `onAttachPlane`.
- Every other reader gets:
  - the plane line, which speaks the switcher's words, is quiet when `ok`, and otherwise carries its own door to System;
  - one door per keeper view (Fleet, Observatory, System), each named by the question the view answers.
- The styles move to `resources/scss/src/apps/agentos/home/Container.scss`, and the lede declares `--fm-font-sans`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `shellPlaneConfigured` (Viewport provider data, default `null`, `apps/agentos/view/Viewport.mjs`) | this ticket; the writer is `ViewportController#mountPlaneSetup` | The controller publishes `false` for a packaged shell without a plane and `true` for one with a plane. Home binds it: `false` shows the product line, the lede and *Connect a plane*. | `null` for a browser build, an unpackaged shell and a failed read (`readPlaneStatus()` resolves `null`), so an unknown status never offers setup. After an attach, the broker relaunches the shell, and the next boot publishes `true`. | JSDoc on the controller method and `AgentOS.view.home.Container` | `ViewportController.spec.mjs` (a browser, an unpackaged shell, a configured shell and a failed read create nothing); `homeContainer.spec.mjs`; the visual driver `homeState.driver.mjs` |
| `instanceState` (Viewport provider data, default `'off'`) | the chrome switcher's verdict | The cockpit's `instanceState` formula writes the ongoing verdict (`cockpit/StateProvider.mjs`: banner hidden → `ok`, degraded → `limited`, else `off`). The instance switch writes `starting`, then `off` on a failed switch. Home's plane line is quiet at `ok` and names every other state. | Home binds the key and never writes it. | JSDoc on the formula and on Home | `homeContainer.spec.mjs` (every word, and an unknown key reads as not connected); the visual Home cases |
| `INSTANCE_STATE_WORDS` (export of `view/fleet/instances/SwitcherButton.mjs`) | the chrome switcher | One vocabulary for the chrome dot and Home's plane line: `ok` connected · `limited` degraded · `off` not connected · `starting` switching. | An unknown key reads as `off`'s word through an own-key guard (`Object.hasOwn`), so `toString` and its kin never leak. | JSDoc on the export | `homeContainer.spec.mjs` |

## Acceptance Criteria

- [ ] AC-1 A packaged shell without a plane sees the headline, the lede and one action, *Connect a plane*, which opens the plane-setup card. It sees no doors and no plane line. A browser build and a shell with a plane publish `null` / `true` and never see this action. `[L4-deferred]` The installed-shell run belongs to #12's installed acceptance, on the first installed build that carries this change. The in-repo proof is the controller and Home unit arms and the visual driver.
- [ ] AC-2 Every other reader sees one door per keeper view (Fleet, Observatory, System). Each door routes to its view and is named by the question that view answers.
- [ ] AC-3 The plane line binds `instanceState` and speaks the chrome switcher's words. It is quiet when the verdict is `ok`; otherwise it names the state and carries a door to System.
- [ ] AC-4 The lede declares the display line's family. A computed-style check pins it and fails without the declaration.
- [ ] AC-5 Visual goldens exist for a returning reader before any answer (dark), a returning reader over a live fleet (both skins) and the first run (both skins). The design owner reviews them.

## Out of Scope

- The team line, and moving the roster read up to the Viewport provider. The team line then takes the headline's slot for returning readers. Both stay on #244, decided there by Clio (the roster read) and Grace (the slot).
- The canvas (#244 AC-1/AC-2).
- A door to Catch Up, which is a cockpit tab with no route.
- *Add your first agent* on Home. While the roster is empty, the cockpit's empty state owns it (#12 AC-4 §1).
- The setup wizard's door. The operator ruled on neomjs/neo#18965 that v1's first run provisions through a setup wizard, so Home's first action for a shell without a plane becomes the wizard's door, with *Connect a plane* beside it as the second. That door joins Home when the wizard has a route.

## Avoided Traps

- **Landing `instanceState` in a visual spec.** The cockpit's formula owns the value and rewrites it on its next tick. The goldens reach its states through the cockpit instead: a cold boot, then a landed fleet.
- **A state dot beside the plane word.** `StateDotComponent` names itself in session words ("working", "rate-limited"), the vocabulary the switcher keeps out of connection states. The word carries the state instead.

## Related

#244 · #12 · #9 · #237 · #318

Live latest-open sweep: the latest 20 open issues at 2026-09-30T09:13Z; no equivalent (newest #340). A2A claim sweep (all read states, last 30): the only Home claim is mine on #244 (`MESSAGE:69a93465`). Memory Core rationale sweep: no prior decision beyond #244's definition and #12's AC-4. Own-assignment sweep: #244 (the parent of this split) and #247.

Decision Record impact: none.

Origin Session ID: 3589c87d-68f1-474c-94c7-b9cbc38b1f5d
Retrieval Hint: "Home first run Connect a plane, plane line, doors, shellPlaneConfigured"




## Timeline

- 2026-09-30T09:14:12Z @neo-opus-vega added the `enhancement` label
- 2026-09-30T09:14:12Z @neo-opus-vega added the `agent-os` label
- 2026-09-30T09:14:12Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-30T09:14:12Z @neo-opus-vega added the `ai` label
- 2026-09-30T09:14:12Z @neo-opus-vega added the `design` label
- 2026-09-30T09:14:35Z @neo-opus-vega added parent issue #9
- 2026-09-30T09:14:35Z @neo-opus-vega added this to the **FM v1** milestone
- 2026-09-30T09:16:22Z @neo-opus-vega cross-referenced by PR #342
- 2026-09-30T09:17:18Z @neo-opus-vega cross-referenced by #244
- 2026-09-30T12:13:55Z @neo-opus-vega cross-referenced by #347
- 2026-09-30T12:32:57Z @neo-fable-clio cross-referenced by #349
- 2026-09-30T13:19:29Z @neo-fable-clio cross-referenced by #351
- 2026-09-30T14:14:35Z @neo-opus-vega referenced in commit `fbf1793` - "chore(agentos): merge dev into the Home first-run branch, visual stamp re-taken on the merged tree (#341)"
- 2026-09-30T14:32:41Z @neo-opus-vega cross-referenced by #358
- 2026-09-30T14:55:53Z @tobiu referenced in commit `731cff1` - "Merge pull request #342 from neomjs/vega/341-home

feat(agentos): Home offers Connect a plane on first run, and doors for the team (#341)"
### @neo-opus-vega - 2026-09-30T14:56:26Z

Landed via #342 (merged by @tobiu at 2026-09-30T14:55:52Z; Emmy APPROVED at fbf1793, review 5367626836; Grace's design sign-off on the goldens). AC-1's installed-shell run stays `[L4-deferred]` under #12's installed acceptance, as the body records. The PR targets dev, so its `Resolves` keyword did not close this ticket.

— Vega (Opus 5.5, Claude Code) 🌿

- 2026-09-30T14:56:28Z @neo-opus-vega closed this issue
- 2026-09-30T15:05:55Z @neo-fable-clio cross-referenced by #361
- 2026-09-30T15:08:28Z @neo-fable-clio cross-referenced by PR #363
- 2026-09-30T18:54:56Z @neo-gpt cross-referenced by PR #360

