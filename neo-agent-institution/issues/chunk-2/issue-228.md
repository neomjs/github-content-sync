---
id: 228
title: 'Brain pin 3 — dev@1ac9492: the fleet allowlist admits fleetGoldenPath'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - dependencies
assignees:
  - neo-fable-clio
createdAt: '2026-09-26T07:21:11Z'
updatedAt: '2026-09-26T07:21:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/228'
author: neo-fable-clio
commentsCount: 0
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
---
# Brain pin 3 — dev@1ac9492: the fleet allowlist admits fleetGoldenPath

## Context

The operator's 2026-09-26 goal: the Golden Path textual view and the graph inside the Fleet Manager. Both panes are on `dev` (#215 → #210, the text pane; #216 → #213, the route graph) and bind the cockpit's `goldenPathEnvelope` leaf, written by `ReadingSurfacesController#loadGoldenPath` from the Brain's `fleetGoldenPath` wire read (neomjs/neo-agent-brain#496 → PR neomjs/neo-agent-brain#499, merged 2026-09-25 19:25Z; the local plane runs cd74d13, which carries it). In the shell and in the browser the panes still render `Unavailable · fleet golden path verb not wired` (Ada's Neural Link read of the operator's packaged shell, 2026-09-25 19:58Z; Grace's #210 note names "the Institution Brain bump for AC-2" as the follow-through).

## The Problem

`package.json` pins `neo-agent-brain` at `8e09275` (the neomjs/neo-agent-brain#377 launch-owner merge, moved there by #171). That commit's `src/fleet/contract/wire.mjs` `FLEET_WIRE_METHODS` ends at `fleetDeploymentState`; Brain `dev` (1ac9492, 192 commits later) adds `adoptAgent`, `releaseAgent` and `fleetGoldenPath`. The cockpit never reaches the plane with a method the installed contract does not list:

- `apps/agentos/fleet/installFleetBridge.mjs:9` imports the contract from `node_modules/neo-agent-brain/src/fleet/contract/index.mjs`, so the bridge it publishes has no `fleetGoldenPath` function in either topology (browser dev server or packaged shell);
- `apps/agentos/view/fleet/cockpit/ReadingSurfacesController.mjs:184` therefore takes the `fallback('fleet golden path verb not wired')` arm — the string both panes show;
- in the packaged shell `harness/main.mjs:463` hands the same pinned list to `harness/fleetCapability.mjs:142-167`, whose `wireMethodSet` refuses the request before the plane (or the bundled Brain) is asked.

Every Golden Path e2e and unit arm lands fixture envelopes through `writeGoldenPath`, never through the wire, which is why the panes are green on `dev` while the live read has never run. #210 AC-2 and #213 AC-5 (the live-plane witness) are unreachable until the pin moves.

## The Architectural Reality

- The installed public contract is the app's one source of the wire vocabulary (#43: "consume the installed public contract"); the app keeps no twin list any more (`apps/agentos/config/fleetWireMethods.mjs` is gone).
- The pin-to-dev contract drift is small and additive — `git diff --stat 8e09275 origin/dev -- src/fleet/contract package.json`: `index.mjs` +1 (exports the new `launchAuthority.mjs`, 23 lines), `wire.mjs` +3 methods, `package.json` 12 lines (dependency bumps; the Brain's engine tarball moves `17b59aad8f` → `7f16355a9b`).
- The Institution forces `overrides['neo.mjs']` to the product engine pin (`harness/pack.mjs` `buildOrganismManifest`, #196), so the bundled Brain runs on `87ac80a6` (engine pin 11). Brain `dev` names `7f16355a9b`, of which `87ac80a6` is an ancestor (15 engine commits between them and engine dev); whether the Brain at 1ac9492 needs anything newer than 87ac80a6 is what the packaged smoke measures.
- CI's "Explicit Brain contract" job (`.github/workflows/ci.yml:63-109`) runs `brain.spec.mjs` + `pack.spec.mjs` with `NEO_AGENTOS_RUNTIME_ROOT` bound to a Brain checkout — those arms plus the unit suite are the mechanical gate.
- Vega's local plane runs Brain cd74d13 (2026-09-25 21:36Z); 1ac9492 is three merges later (neomjs/neo-agent-brain#520, #501, #510). The plane's recreate onto the pinned head is her call under neomjs/neo-agent-brain#64, not this ticket's.

## The Fix

1. `package.json`: `neo-agent-brain` → `github:neomjs/neo-agent-brain#1ac9492…` (dev head at filing); `package-lock.json` follows from `rm -rf node_modules/neo-agent-brain && npm install` (a stale github-SHA install survives a plain `npm install`).
2. Prove the gate moved: the installed `wire.mjs` lists `fleetGoldenPath`; the browser cockpit against the host dev fleet server renders the plane's real Golden Path state; the packaged shell in plane-attach renders the same (the #210 AC-2 / #213 AC-5 witness).
3. The harness contract specs with `NEO_AGENTOS_RUNTIME_ROOT`, the unit suite, the visual suite (a Brain pin re-renders no golden by construction, so a moved golden is a finding, not a regolden), and the packaged smoke (`profileMode packaged-product`, bundled Brain up + cleanStop) on the rebuilt `.app`.
4. The engine pin stays at 87ac80a6 unless the smoke proves the Brain at 1ac9492 needs `7f16355a9b`; then engine pin 13 rides in the same PR with its own drift line (precedent #195).

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `FLEET_WIRE_METHODS` as installed (`node_modules/neo-agent-brain/src/fleet/contract/wire.mjs`) | neomjs/neo-agent-brain `src/fleet/contract/wire.mjs` at the pin | lists `adoptAgent`, `releaseAgent`, `fleetGoldenPath` | none — the bridge publishes exactly the list; a missing method is the `verb not wired` fallback the panes already render | `installFleetBridge.mjs` docblock, unchanged | AC-1 |
| `bridge.fleetGoldenPath({})` from `ReadingSurfacesController#loadGoldenPath` | Brain #499's envelope `{capability, admission, route, rem, sources}` | the panes render the plane's state through `GoldenPathEnvelope.fromWire` | the fallback envelope (unavailable + reason), as today | #210 / #213 bodies | AC-2, AC-4 |

## Decision Record impact

aligned-with ADR 0034 §2.3.4 — the fleet allowlist is the capability-allowlist discipline; this moves the list's source commit, not the discipline.

## Acceptance Criteria

- [ ] AC-1 The installed contract's `FLEET_WIRE_METHODS` lists `fleetGoldenPath` (read from `node_modules` after the install); `package.json` and `package-lock.json` name the same Brain SHA.
- [ ] AC-2 The browser cockpit (`preview_start institution` + the host dev fleet server from a Brain checkout at the pin) shows the Golden Path pane's currency line from the wire, not the `verb not wired` fallback — a Neural Link read of the provider leaf's `capability.state`.
- [ ] AC-3 Unit suite green; `brain.spec.mjs` + `pack.spec.mjs` green with `NEO_AGENTOS_RUNTIME_ROOT`; visual suite green with no golden moved; the packaged smoke green on the rebuilt `.app`.
- [ ] AC-4 (post-merge) The packaged shell in plane-attach on the team machine renders the plane's Golden Path state in both panes — the #210 AC-2 / #213 AC-5 witness, receipts on those tickets.

## Out of Scope

The engine pin (unless AC-3 forces it); the plane recreate onto the pinned head (Vega, neomjs/neo-agent-brain#64); the 3D observatory (neo#10034's promotion); any change to the panes themselves.

## Related

#210, #213, #215, #216 (the panes) · #43, #171 (the two prior Brain pins) · #195 (the engine-pin precedent) · neomjs/neo-agent-brain#496, neomjs/neo-agent-brain#499 (the read) · neomjs/neo-agent-brain#64 (the plane).

Live latest-open sweep: the latest 20 open issues of this repository at 2026-09-26T07:18:32Z — no equivalent (#227 is the PAT window, #225 the refusal banner). A2A in-flight sweep at 07:18Z: no claim on the Brain pin (Grace's #210 note names the bump as a follow-through; the morning's claims — Mnemosyne's #19232, Ada's #227, Grace's tear-out lane — are elsewhere). Memory Core sweep (`query_raw_memories`): Mnemosyne's #499 turn and Grace's #215 turn both name this bump as the next step; no ticket. Own-assignment sweep: #10 (the epic) only. Structure map: N/A (a dependency pin, no file placement).

Origin Session ID: 26b775fe-f8d9-4258-809c-09d9e5ef8ed1
Retrieval Hint: `query_raw_memories("Institution Brain pin fleetGoldenPath verb not wired allowlist")`

## Timeline

- 2026-09-26T07:21:11Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-26T07:21:12Z @neo-fable-clio added the `enhancement` label
- 2026-09-26T07:21:12Z @neo-fable-clio added the `agent-os` label
- 2026-09-26T07:21:12Z @neo-fable-clio added the `ai` label
- 2026-09-26T07:21:12Z @neo-fable-clio added the `dependencies` label
- 2026-09-26T07:21:30Z @neo-fable-clio added parent issue #9
- 2026-09-26T07:25:38Z @neo-opus-grace cross-referenced by #229
- 2026-09-26T07:28:22Z @neo-fable-clio cross-referenced by #230

