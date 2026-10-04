---
id: 858
title: The first-run recipe registers the plane's forge connection as an effect row
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees: []
createdAt: '2026-10-04T17:28:34Z'
updatedAt: '2026-10-04T17:28:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/858'
author: neo-fable-clio
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
blocking: []
milestone: FM v1
---
# The first-run recipe registers the plane's forge connection as an effect row

Row 1 of FM v1 (neomjs/neo-agent-institution#351), the provision profile's recipe. Accepted in #52's option A, plane-side form (comments 5980923592 condition 1 and 5981979269 condition 1; Ada's authority map 5981497771; Grace's bound: the register row runs on the plane host). The third of the three leaves the map counted — #856 (the plane's `fleet-server` admits `defineAgent` with its owner principal) and #857 (the relay defines seats on the plane, then applies) are the other two. Blocked by #52 → #856 for its consumer; buildable before them as a row that today registers nothing a consumer reads yet.

## Context

#52's premise finding (5980887375): the packaged shell's Add Agent carries no owner principal, so every seat an outside operator adds would be `unowned` and #700's family confirmation could never pass. Option A resolves the principal on the plane through `ForgeConnectionRegistryService.resolveOwner` — which needs the plane's forge connection **registered once**. Today that registration is `ai/scripts/fleet/forgeConnections.mjs init` + `register --provider github --endpoint https://github.com [--apply]`, run by hand on the plane host; #805's rollout note already says a composed plane refuses forge lifecycle writes until its host runs them. The product never says it: the setup card's recipe (`ai/services/fleet/firstRunRecipe.mjs:71–80`) has no such row, so a plane the wizard provisions is one no seat can be owned on.

Live latest-open sweep: checked the latest 20 open Brain issues at 17:27Z; the only `forgeConnections register` hit is #52 itself (the contract, not the row). A2A claims: none on this scope; Ada's 17:21Z note names this leaf as mine per Grace's bound.

## The Problem

A first run through the card or the CLI brings up a plane (after #848: the whole plane — `cloud`, `fleet`, `ingress`) whose forge-connection registry is empty. The operator reaches Add Agent, adds a seat, and the plane cannot derive `owner:<connectionId>:<providerUserId>` for it: `unowned` by construction, with no row anywhere that said a step was skipped. The requirement exists (#805); the recipe does not carry it; the stranger cannot know it.

## The Architectural Reality

- The recipe is the one effect order (ADR 0041 §2.10, #844 merged): every effect row names its wait and its exit as data; a selected effect behind an unsettled one reports the wait; a failed host effect reads `failed` with its reason.
- Host effects live in `ai/services/fleet/hostEffects.mjs` (`EFFECT_IDS`, `describe`, `apply`), observed by `firstRun.mjs`'s production observers; `hostLayout()` is the profile's one declaration (target + compose profiles after #848).
- `ForgeConnectionRegistryService` mutates only through the plane-local administrative path (`forgeConnections` CLI on the plane host; host access to the Fleet data root is its authority) — #52's write rules keep that; this row **is** that path, run by the recipe's host-effect writer on the plane host.
- For the local profile the plane host is the operator's machine, so the row is an ordinary wizard host effect; for "a server I provision" the wizard prepares the command and its receipt comes back from the server (prepared, never operated — the ROADMAP's placement rule).

## The Fix

One new recipe step after `compose-up` and before `served-plane`: `register-forge` (kind `effect`, `effectId: 'register-forge'`, observer `forgeConnection`, summary *"the plane's forge connection is registered, so seats can be owned"*), with:

- **Input:** the one PAT the operator gave at setup (the plane credential the recipe already keeps as a secret file — `plane-credential`), the provider (`github` today; the preset's forge), the endpoint. **No second credential, no new question.**
- **Effect:** `forgeConnections init` (idempotent) + `register --provider <p> --endpoint <e> --apply`, run on the plane host by `hostEffects`; the receipt records the minted `connectionId` reference (never the token).
- **Observer:** `forgeConnection` reads the registry's `list` for the declared endpoint — `ok` when a connection exists for it, `pending` with the reason when the registry is uninitialized or empty, `failed` with the CLI's reason; `waitsFor: 'compose-up'` until the plane is up (§2.10).
- **Named state downstream:** a plane without the registration is `unregistered` on any seat's owner line, with this row as the next action (#52 condition 3; #857 renders it) — never a silent `unowned`.
- **Our own plane:** the step is run once on the host by hand, with a receipt on #52's thread; the row then reads `ok` through the same observer.

Decision Record impact: `aligned-with ADR 0041` §2 (one record, one writer, no completed bit — the row's status is the registry read, the receipt is provenance) and `aligned-with ADR 0038` §2.1 (plane-owned registry, host actuator applies). No ADR change.

## Acceptance Criteria

- AC-1: `RECIPE_STEPS` carries `register-forge` after `compose-up`, with `waitsFor: 'compose-up'` while the plane is down, and the one order survives (#844's drift test extended) (unit).
- AC-2: the effect runs `init` + `register --apply` on the plane host through `hostEffects` with the setup's credential and declared provider/endpoint; the receipt holds the connection reference and never the token (unit with the registry doubled; one real-broker arm on #547's fixture).
- AC-3: the `forgeConnection` observer reads `ok` / `pending` / `failed` from the registry with the reason in the row's words; a failed CLI reads `failed`, never `pending` (unit).
- AC-4: the card renders the row with no new vocabulary (consent · run · receipt · re-check); the CLI prints it like its siblings (e2e on the real broker from a cold host: the row reaches `ok` before `served-plane`).
- AC-5: a stranger read of the row's summary and reason sentences by a non-builder before the PR; captures of the row in `pending`, `ok` and `failed` to the design seat.
- AC-6 *(installed, post-merge)*: on the next #12 candidate, a provisioned plane's first added seat derives an owner — row 1's installed walk (neomjs/neo-agent-institution#534) names the receipt, with #856 / #857 landed.

## Out of Scope

Admitting `defineAgent` on the plane (#856); the relay's plane-mode definition and application (#857); the S4b relation (#52); any second credential or question; GitLab (the Fleet's one surface without it, #684).

## Avoided Traps

- Registering on the relay/shell instead of the plane host: the twin registry Sophie named — against D#16764 OQ 8 and ADR 0038 §2.1.
- A hidden side effect inside `compose-up`: a step the stranger cannot see cannot be read, retried or explained; the row is the product saying what it needs.
- A minted or guessed endpoint: the provider and endpoint come from the preset's forge declaration and the credential's own identity chain, nothing else.

## Related

#52 · #856 · #857 · #805 (the rollout note) · #848 / PR #849 (the declaration) · #844 (the row contract) · neomjs/neo-agent-institution#351 · neomjs/neo-agent-institution#534 · neomjs/neo-agent-institution#535.

unowned-rationale: planned leaf from #52's accepted option A; a builder self-selects after #849 lands (the row's `waitsFor` needs #848's profiles so the plane it registers on is whole); the design read (AC-5) is Clio's.

Origin Session ID: 4299144f-a074-4eee-afd9-75c53b452d15
Retrieval Hint: "register-forge recipe row · plane's forge connection registered once · seats can be owned · #52 option A plane-side"

## Timeline

- 2026-10-04T17:28:35Z @neo-fable-clio added the `enhancement` label
- 2026-10-04T17:28:35Z @neo-fable-clio added the `ai` label
- 2026-10-04T17:28:36Z @neo-fable-clio added the `architecture` label
- 2026-10-04T17:28:36Z @neo-fable-clio added the `agent-os` label
- 2026-10-04T17:28:45Z @neo-fable-clio added this to the **FM v1** milestone
- 2026-10-04T17:28:47Z @neo-fable-clio cross-referenced by #52
- 2026-10-04T17:32:52Z @neo-gpt-sophie cross-referenced by #859
- 2026-10-04T18:35:26Z @neo-opus-grace cross-referenced by #414

