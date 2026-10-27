---
id: 726
title: The cockpit says when the plane cannot own seats yet
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-gemini-julia
createdAt: '2026-10-17T18:36:34Z'
updatedAt: '2026-10-17T21:15:07Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/726'
author: neo-fable-clio
commentsCount: 0
parentIssue: 477
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-17T21:15:07Z'
---
# The cockpit says when the plane cannot own seats yet

## Context

Tonight's walk (2026-10-17 ~20:10 Berlin): Add an agent for `neo-gemini-maryam` asked for the PAT, then printed the plane's refusal — `'defineAgent' is a lifecycle-write verb and requires a forge-resolved admission subject — the owner is uninitialized: the plane's connection registry is not initialized: the plane's setup step 'register-forge' initializes it`. Read from the plane: `forgeConnections.mjs status` → `state: absent`, declared github / https://api.github.com (compose project `neo-local-agent-os`, fleet-server at Brain d628999). The cause is an operational step never run: Brain PR #805 (merged 2026-10-03) says in its own body that a composed plane refuses every forge-authenticated lifecycle write until its host runs `init --apply` and `register --provider github --endpoint https://api.github.com --apply` in the `fleet-server` container, and lists exactly that as an unchecked Post-Merge Validation item (Residual-Owner Brain #83); the plane was re-cut on 2026-10-13 and nobody ran it. The packaged Fleet Manager's local lifecycle never resolves an owner (#805's own words), which is why the nine seats run while a plane write fails. Emmy's installed reproduction: #505 (comment 6238570247). The operator's unblock tonight is the pair itself, in his hands. Mnemosyne's call on the shape (A2A 591b1676): the Connect-door route (adopt a running plane into a run) is an ADR 0034 / 0041 amendment under #351, deferred; **this leaf is the cheap half — the cockpit reads what the plane already says.**

## The Problem

The plane already knows and says it: `ForgeConnectionRegistryService#resolveOwner` → `{state: 'uninitialized', reason}` rides every request context (Brain `fleetServer.mjs:142`) and `GET /fleet/probe` echoes it as `identity` (`fleetServer.mjs:384–394`). The cockpit reads none of it (no `fleet/probe` reader under `apps/` or `src/`), so the operator learns that the plane cannot own seats only after typing a token into a sheet, from a refusal paragraph in red. Row 2's rule (#477): every surface names its state with a reason and a next step — here the state has been nameable since October 3.

## The Architectural Reality

- Brain `fleetServer.mjs:121–145` `createFleetRequestContext`: a forge-authenticated context carries `ownerPrincipal` (admitted) or `ownerResolution {state, reason}` (`uninitialized` | `unavailable` | `unregistered` | `refused`); `:384–394` `GET /fleet/probe` → `{identity: context, fleetDataDir, pid, startedAt}`; `fleetServerPolicy.mjs:194` the refusal text. No new Brain verb is needed for a read (Mnemosyne: a Brain leaf only if a wire verb is preferred over the probe).
- Institution `apps/agentos/view/fleet/instances/AddAgentForm.mjs`: `onSubmitClick` (`:337`) → the token sheet → `defineAgent`; the status line `fm-add-status` (`:344`, `:464`) prints the outcome after the fact.
- The place the plane is named (the Add form's `Agent OS: 127.0.0.1:3102` line; the cockpit's plane status in the overview / roster header) is where one state line belongs.
- The PAT itself is unaffected: possession admits the transport, identity owns the records (Brain #783's ledger).

## The Fix

1. **One plane-admission read:** `GET /fleet/probe` → `identity.ownerPrincipal` present = the plane can own seats; `identity.ownerResolution` present = it cannot, with the plane's own `reason`. Re-read on Add and on the plane line's refresh cadence; never cached across a plane change (the probe carries `pid` / `startedAt`).
2. **The Add form reads it before the token prompt:** while the plane cannot own seats, the submit opens no token sheet; the status line says `This plane cannot own seats yet — <reason>. On the plane host: init, then register (the setup step 'register-forge'); then Add again.`, with the two commands behind a *Show the commands* disclosure (copyable; the compose project and service come from the plane's own probe / the local-agent-os README, never typed by the operator). The submit glyph question is #724's.
3. **One plane-state line where the plane is named:** `cannot own seats yet · register-forge` while `ownerResolution` is set; nothing when admitted. The same words as the Add form, one source.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Add form, before the token sheet | Brain `GET /fleet/probe` `identity` (`fleetServer.mjs:384`) | no sheet while `ownerResolution` is set; the reason and the pair in the status line | today's post-token refusal | CARD-CONTRACT status-line words | component arm AC-1 |
| Plane-state line (overview / roster header) | the same probe | `cannot own seats yet · register-forge` while unresolved; absent when admitted | nothing | the Row 2 words (#477) | unit + visual AC-2 |
| Probe unreachable | — | the form proceeds as today (the plane's refusal stays the backstop); the line reads `plane admission unknown` | — | — | component arm AC-3 |

Decision Record impact: aligned-with ADR 0034 (the plane says its state; the shell renders it) — no amendment here; the Connect-door run is #351's deferred amendment.

## Acceptance Criteria

- [ ] AC-1 component: with a probe answering `ownerResolution: {state: 'uninitialized', reason}`, submit opens no token sheet and the status line carries the reason and the two commands; with `ownerPrincipal`, today's flow is unchanged.
- [ ] AC-2 unit + visual: the plane-state line renders the words while unresolved and nothing when admitted; dark + light goldens.
- [ ] AC-3 component: a probe that cannot be read keeps today's submit path and says `plane admission unknown` on the line.
- [ ] AC-4 post-merge, installed: on the next #12 candidate against a plane with an absent store (a fresh compose project before `init`): Add says it before the token; after the pair, the line disappears and Add proceeds — the Row 1 walk (2026-10-23) or a receipt on this ticket. **[L4-deferred — operator handoff needed: the Row 1 walk, 2026-10-23, on a candidate at or after PR #729]**

## Out of Scope

The Connect door running the recipe against an attached plane (#351, deferred; the ADR 0034 items 8/10 + 0041 amendment first — Mnemosyne); a Brain wire verb for admission (the probe suffices); a remote mutation route (none in v1 by Brain #783's ledger); the form's three UI defects (#724); the migration itself (Brain #805's Post-Merge Validation, Residual-Owner Brain #83 — ticked once the host runs the pair).

## Avoided Traps

Running `init` / `register` from the cockpit (host access to the Fleet root is the authority; v1 has no remote mutation — Brain #783); caching admission across plane restarts; inventing words (the plane's reason sentence is the operator's sentence); a modal (the status line and a disclosure keep the frame usable, as the setup card's spec says).

## Related

#477 (parent, Row 2), #505 / #724 (the form's design), #351 (the Connect-door amendment, deferred), #475 (the setup card), Brain #783 / PR #805 (the admission guard and its rollout step), Brain #83 (that step's residual owner), Brain #732 (the recipe), #505 comment 6238570247 (Emmy's installed reproduction), defect-note f90bcb2d.

Live latest-open sweep: checked the latest 20 open Institution issues at 2026-10-17T18:24Z and the newest 8 again at 18:3xZ (newest #725 / #724, mine); none on plane admission. Exact search `register-forge OR "forge connection"` (open, org-wide): none. A2A in-flight claim sweep (last hour): Mnemosyne's answer assigns this half to me (591b1676); Emmy's receipt (b9564090) and Vega's offer (447bf4fd) name no competing leaf. MC sweep: the defect's own trail only. Own-assignment sweep: #505 / #507 / #351 — #351 is the deferred neighbor, not an overlap. Structure map: N/A (Institution).

handoff: @neo-gemini-pro first refusal (Vega declined 18:37Z — the 3D engine keeps his seat — and named her: Brain #1082 shipped on the same plane path); the design read and the acceptance stay with me; unowned until she answers.

Origin Session ID: cd7bf4ee-8722-43be-baf8-29f9ed658722

Retrieval Hint: "cockpit plane cannot own seats yet register-forge probe identity ownerResolution Add form before the token prompt"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session cd7bf4ee-8722-43be-baf8-29f9ed658722



## Timeline

- 2026-10-17T18:36:35Z @neo-fable-clio added the `enhancement` label
- 2026-10-17T18:36:35Z @neo-fable-clio added the `agent-os` label
- 2026-10-17T18:36:36Z @neo-fable-clio added the `ai` label
- 2026-10-17T18:36:36Z @neo-fable-clio added the `design` label
- 2026-10-17T18:36:58Z @neo-fable-clio added parent issue #477
- 2026-10-17T18:38:22Z @neo-fable-clio cross-referenced by #724
- 2026-10-17T18:43:00Z @neo-fable-mnemosyne cross-referenced by #727
- 2026-10-17T18:50:50Z @neo-fable-clio cross-referenced by PR #805
- 2026-10-17T19:33:04Z @neo-fable-clio cross-referenced by PR #728
- 2026-10-17T19:41:02Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-17T20:22:55Z @neo-gemini-julia cross-referenced by PR #729
- 2026-10-17T20:54:16Z @neo-gemini-julia referenced in commit `513bf19` - "fix(agentos): wire packaged probePlane relay and tighten register endpoint (#726)"
- 2026-10-17T21:15:07Z @tobiu referenced in commit `1a71938` - "feat(agentos): cockpit plane admission read and pre-token-sheet gate (#726) (#729)

* feat(agentos): cockpit plane admission read and pre-token-sheet gate (#726)

* fix(agentos): wire packaged probePlane relay and tighten register endpoint (#726)"
- 2026-10-17T21:15:07Z @tobiu closed this issue

