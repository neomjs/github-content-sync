---
id: 430
title: Carry the wizard backend and scroll edge in the next Fleet package
state: CLOSED
labels:
  - enhancement
  - ai
  - build
  - dependencies
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-02T09:36:14Z'
updatedAt: '2026-10-02T12:02:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/430'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 429 The engine''s scrollEdge reaches the cockpit: the mailbox drops its interim body, the memories pane requests at the edge'
closedAt: '2026-10-02T11:35:21Z'
---
# Carry the wizard backend and scroll edge in the next Fleet package

## Context

The next Fleet package must consume the merged first-run backend and grid edge event. Pin 10 is in #408 / PR #423 and remains Ada's work. This is the bounded successor pin and package preparation owned by Emmy, including the atomic retirement of the interim mailbox Body supplied by Vega; #384 owns the wizard renderer, #429 the memories scroll-edge consumer, and #12 the installed-update witness.

## The Problem

At PR #423's pin, Brain `cbd11cb62be316ec9f195b6dff0732e0cfe8ba40` does not contain the first-run recipe or file-backed credential step. Engine `08ff2a55e6f851f1f2002f5dd72686ecc6cce779` lacks `scrollEdge`. Repackaging those pins cannot deliver either capability.

The frozen successors are both merged descendants:
- Brain `f9ccc2e260932e150c86ce1fc301d649b70aed8f`: three merges after pin 10 — neomjs/neo-agent-brain#732, #739 and #736.
- Engine `93769448934166a8c98b4d99eccda4c3d347caeb`: three merges after the current engine pin — neomjs/neo#19352, #19355 and #19357.

## The Architectural Reality

`package.json` and `package-lock.json` own the installed dependency identity; `.github/workflows/ci.yml` independently selects Brain for the cross-repository contract and must agree. The Engine lock entry contributes to `buildScripts/checkVisualBaselines.mjs`'s input stamp. The existing harness packer builds the product with explicit Brain and Engine inputs.

The Brain delta changes no `src/**` or package dependencies. The Engine delta includes dock disposal/retry and the grid event; its package change is the cssnano devDependency patch. No SCSS changes occur in either upstream delta. The hosted graph-provider limitation remains separately owned by neomjs/neo-agent-brain#744; this pin does not establish a supported hosted first-run witness.

## The Fix

After pin 10, advance the two dependency refs, regenerate the matching lock entries, align CI's Brain ref and verify the visual baseline input identity against the chosen Engine. Run the existing Institution contract checks, including the affected dock/grid and packaged boundaries. Adapt only integration fixtures whose failure is empirically established during these checks. Prepare a package with an exact source/hash/smoke receipt for #12.

Prescription checked: `package.json`, `package-lock.json`, `.github/workflows/ci.yml` — these own consumer dependency selection. Copying upstream implementations into the cockpit would create a second owner.

## Contract Ledger

| Surface | Authority | Behavior | Boundary | Evidence |
|---|---|---|---|---|
| Brain dependency + CI checkout | root manifests and existing CI | all name `f9ccc2e260932e150c86ce1fc301d649b70aed8f` | lands after #423; no independent configuration changes | lock/ref census and cross-repository contract |
| Engine dependency + mailbox retirement | root manifests; native `Neo.grid.Body` event | both manifests name `93769448934166a8c98b4d99eccda4c3d347caeb`; mailbox relays one native edge event | memories consumer adoption stays #429 | existing once-per-count mailbox controls, focused grid/dock and visual checks |
| Packaged candidate | existing harness packer | receipt binds product, Brain, Engine and artifact hash | a build or isolated smoke is not installed acceptance | isolated packaged smoke and #12 handover |

Decision Record impact: aligned-with existing dependency ownership and ADR 0041; no new authority or config policy.

## Acceptance Criteria

- [x] AC-1 Both manifests and CI name the exact intended pins; the lock is installable and the visual input stamp is coherent.
- [x] AC-2 Existing isolated and Brain-bound contract checks pass, with relevant Darwin visual/NL evidence and explicit attribution of any independently reproduced pre-existing failure.
- [x] AC-3 A packaged candidate from the reviewed change has a receipt with exact source pins, SHA-256 and isolated-smoke result, handed to #12. Installed acceptance is retained on #12, not claimed by this leaf.

## Validation fixture correction — 2026-10-02

The full Neural Link battery passed 53/54; the transport-kill journey timed out after 150 seconds. Its existing `startLivenessFleetServer().close` only calls `server.close(resolve)`, which gracefully waits for open event streams. A disposable Node 24.19 HTTP/event-stream control kept that promise pending until `server.closeAllConnections()` ended the owned connection. The fixture is byte-identical to merged `dev` (SHA-256 `89ea5829af5f25c41f915a97961a007f8c2af30c8d73801eeff77d16446c3f73`).

Scope includes the narrow fixture correction in `test/playwright/e2e/agentos/FleetCockpitLivenessNL.spec.mjs`: close the listener and its owned active HTTP connections, so the advertised transport death actually occurs. Re-run both liveness journeys. No production transport or global process cleanup changes.

## Packaged first-paint sampling correction — 2026-10-02

The first package built at `3169c692` booted both windows, its isolated Brain, assets and shared heap, and stopped cleanly, but its one-shot first-paint observer captured `rosterState: cold`, an empty label and `emptyCta: true` at 1.5 seconds. The later popup reported the live empty roster. Main correctly rejected the first inconsistent tuple.

Scope includes a bounded readiness correction in `harness/preload.cjs`: an empty CTA counts as an answer only once its head has left cold; a cold report after the settle window requires no CTA. The strict main-process coherence verdict and timeout stay unchanged. Three red-first unit controls cover the transition with either label and a transition that never settles. This does not change the cockpit, admit a new state or replace the first-persistence work.

## Atomic mailbox retirement — 2026-10-02

After rebasing onto merged #420, both CI contract jobs and the local `container.spec.mjs:542` control reproduce two identical edge events where one is required. The interim mailbox Body calls the newly pinned native implementation, then fires its own event. Its own retirement contract binds removal to this pin.

Vega and Emmy agreed that #433 carries Vega's bounded #429 mailbox commit: delete the interim class, remove Grid's body override and the unused topology-family row, and refresh the visual input stamp. Keep the existing mailbox event controls unchanged; they now exercise the engine primitive through the real grid relay. #429 retains the memories consumer adoption. The pane's in-flight gate is not a substitute for single event ownership.

## Out of Scope

The wizard renderer (#384), memories drain replacement (#429), hosted-provider repair, live plane redeployment, and live installation/seat moves. Those owners and the recorded #12 checkpoint/rollback plan remain intact.

## Avoided Traps

Rebuilding a stale pin and calling merged upstream features delivered; changing only CI or only the manifest; treating a stamp as pixel evidence; modifying Ada's active branch; using prior peer checkpoints for a new app restart.

## Related

#408 / #423 · #12 · #7 · #351 · #384 · #429 · neomjs/neo-agent-brain#732 · neomjs/neo-agent-brain#736 · neomjs/neo#19357.

## Sweeps

Live latest-open and all-read-state A2A sweeps on 2026-10-02 found no successor pin claim; #429 explicitly excludes the pin. Own-assignment scan found no Institution lane overlapping this work. MC queries on stale package/pin delivery recovered the existing #12 handover and pin-8 review, not a successor owner. Brain structure map ran successfully; this changes existing manifest/CI/receipt surfaces, no new module.

Origin Session ID: 3acb1755-5285-4f3a-a74a-dae637bb629d
Retrieval Hint: Fleet next dependency pins wizard recipe scrollEdge packaged candidate after pin10.

🪡 Emmy · GPT-6 Astra · Codex


## Timeline

- 2026-10-02T09:36:14Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-02T09:36:15Z @neo-gpt-emmy added the `enhancement` label
- 2026-10-02T09:36:16Z @neo-gpt-emmy added the `ai` label
- 2026-10-02T09:36:16Z @neo-gpt-emmy added the `build` label
- 2026-10-02T09:36:16Z @neo-gpt-emmy added the `dependencies` label
- 2026-10-02T10:30:50Z @neo-fable-clio cross-referenced by #431
- 2026-10-02T10:39:28Z @neo-gpt-emmy cross-referenced by PR #433
- 2026-10-02T10:48:38Z @neo-opus-vega cross-referenced by #429
- 2026-10-02T10:48:39Z @neo-opus-vega marked this issue as blocking #429
- 2026-10-02T11:02:33Z @neo-gpt-emmy cross-referenced by #12
- 2026-10-02T11:35:21Z @tobiu referenced in commit `f2dd081` - "feat(deps): carry the first-run backend and grid scroll edge (#430) (#433)

* feat(deps): carry first-run backend and grid scroll edge (#430)

* fix(harness): wait for the empty-answer header before first paint (#430)

* refactor(agentos): the mailbox grid rides the engine's scrollEdge, the interim body goes (#429)

With the engine pin carrying neomjs/neo#19357, Neo.grid.Body fires scrollEdge itself; the interim subclass fired it a second time, and the mailbox's once-per-count arm went red on the pin PR. The grid relays the engine's event unchanged; the subclass, its body config and its FAMILIES row are gone.

---------

Co-authored-by: Neo Opus Vega <neo-opus-vega@neomjs.com>"
- 2026-10-02T11:35:22Z @tobiu closed this issue
- 2026-10-02T11:37:44Z @neo-opus-vega cross-referenced by PR #434
- 2026-10-02T11:38:26Z @neo-opus-grace cross-referenced by #414

