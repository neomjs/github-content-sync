---
id: 1543
title: A seat Start launches its family's admitted harness generation
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-09T19:51:06Z'
updatedAt: '2026-11-10T04:04:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1543'
author: neo-opus-ada
commentsCount: 1
parentIssue: 1531
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-10T04:04:59Z'
---
# A seat Start launches its family's admitted harness generation

## Context

This is #1531's seat-side leaf. #1534 admits vendor builds into immutable generations, and #1537 and #1539 fetch new builds into that store. No seat launches from a generation yet: every seat still starts the mutable `/Applications` bundle. This leaf makes a Start pin its family's selected generation, so updates reach seats at their next Start and a rejected build never does.

## The Problem

- `getHarnessBinaryPath(harnessType)` returns `harnessBinaryPaths[type] || AiConfig.fleet.harnessBinaries[leafKey]`, which is always the shared `/Applications` bundle.
- A build that fails admission still sits at that path, so the family cannot start. This is today's Codex outage (#1530).
- No seat record says which vendor build it runs, so the cockpit cannot show "running vX, selected vY".

## The Architectural Reality

- **Where the binary is decided.** The async `startAgentProvisioned` (`ai/services/fleet/startAgentProvisioned.mjs`) resolves the binary once, through `lifecycleService.assertRemoteMcpCapability(agent, …)`. That returns `{harnessType, binaryPath, launchBinaryPath}` from `getHarnessBinaryPath` (`FleetLifecycleService.mjs`). `start()` then binds that snapshot (`opts.remoteMcpCapability`) instead of re-reading AiConfig. This async path is where a generation is selected and, if needed, admitted.
- **The bundled CLI.** `codex-desktop` also resolves the bundled CLI through `getHarnessBinaryPath('codex')` (`FleetLifecycleService.mjs`, auth command and capability check). It must come from the same generation as the app (Julia's #1531 epic review, contract 1).
- **Engine (#1534).**
  - `materializeHarnessGeneration({root, family, sourceBundle, source})` admits a candidate.
  - `selectHarnessGeneration({root, family})` returns `{id, appBundle, binaryPath, shortVersion, buildVersion}` or `null`.
  - `readHarnessGenerationLedger({root, family})` (on `dev` since #1538) returns the admitted and rejected entries. Bootstrap uses it; nothing else parses the ledger.
  - Re-admitting a known build never reorders the ledger.
- **Wakes.** Since #1542 (#1545 on `dev`), a GUI wake resolves its target by the seat's unix pid for every family (`localWakeAdapters.mjs`). Launching from a generation copy therefore needs no wake change here.
- **Precedent.** The runtime-generation pin in the seat runtime record (`seatRuntimeRecord.mjs`, `runtimeGenerationPin`) and ADR 0034 §2.5 item 3.
- **Every reader of the binary path**, census of `getHarnessBinaryPath` and `harnessBinaries` on `dev` `4605d893`, and what each must read for an opted-in family. The census at `2d0f7a6c` missed the three Codex app-server rows, which existed then.

  | Reader (`FleetLifecycleService.mjs` unless named) | Today reads | Must read |
  |---|---|---|
  | `assertRemoteMcpCapability` (both paths) | config | the selection; this is where it is made |
  | `resolveAntigravityStartContext`, `validateAntigravityStartContext` | config | the selection |
  | `resolveLaunch` with the proof | the proof's `launchBinaryPath` | unchanged |
  | `start` / `resolveLaunch` without a proof: a Start with no repository, or with no remote MCP, passes none | config | the selection. The census at `4605d893` said every Start passes the proof; these two paths do not. |
  | `start`, the Codex auth command (`getHarnessBinaryPath('codex')`) | config CLI | the CLI in the pinned generation |
  | `adoptLeasedSeat`, the launch and the auth command | config | the pinned generation. Otherwise a seat running from a generation is not recognised after a Fleet restart. |
  | `desktopProfileInUse` | config binary | every admitted generation of the family. Otherwise a running seat reads as idle and its MCP rows get rewritten under it. |
  | `seatModelCatalog.mjs`, the Codex catalog | config CLI | the pinned generation's CLI for a `codex-desktop` seat |
  | `seatModelCatalog.mjs`, the claude-code catalog | config | unchanged: a CLI family |
  | `ai/daemons/wake/daemon.mjs`, `deliverViaCodexAppServer` | config CLI | the seat's pinned generation CLI. That is a precondition of opting `codexDesktop` in, not part of this leaf: no Fleet pin reaches the wake daemon yet. |
  | `ai/scripts/lifecycle/resumeHarness.mjs`, the `codex-app-server` resume (live only with `RUN_LIVE_CODEX_APP_SERVER=1`) | config CLI | the same, with the same precondition |
  | `ai/daemons/wake/localWakeAdapters.mjs`, the `codex-app-server` branch | the route's `adapterConfig.codexBinary`, written by the operator's `--adapter-config` | unchanged: operator-owned host configuration |

- **Trust boundary.** The seat can write its own lease, so `adoptLeasedSeat` takes only the pid, its start time, `startedAt` and the checkout from it. A pin may enter the lease only as a generation id. It counts only when the Fleet-owned ledger lists that id as admitted for the seat's family, and the binary path always derives from the store root and the ledger entry, never from a lease value. An unknown id makes the lease invalid.

## The Fix

1. **AiConfig, per ADR 0019.** Add a `fleet.harnessGenerations` subtree:
   - `root`: `leaf(path.resolve(os.homedir(), '.neo-ai', 'harnesses'), 'NEO_FLEET_HARNESS_GENERATIONS_ROOT', 'string', {planeMember: false, planeMemberReason: …})`, read at its use sites;
   - one boolean opt-in per GUI family (`codexDesktop`, `antigravity`, `claudeDesktop`), every default `false`. A family that has not opted in behaves exactly as today.
2. **Bootstrap.** In the async Start path, before the catalog or the capability proof reads the harness, for an opted-in family:
   - if the installed bundle (the `.app` containing the `harnessBinaries` path) carries a build the family ledger has neither admitted nor rejected, materialise it once (`source: 'installed'`);
   - then select the generation.
3. **Launch from the selection.**
   - `launchBinaryPath` is the selected generation's main binary.
   - For `codex-desktop`, the bundled CLI is the same relative path inside that generation.
   - With no selection, Start refuses by name and carries the family's latest rejection reason. It never falls back to the rejected `/Applications` copy.
4. **Pin.** The lifecycle record carries `{family, generationId, shortVersion, buildVersion}`, `status()` reports it, and the lease carries only its id. The seat runtime record (`seatRuntimeRecord.mjs`) does not: it accepts exactly its version-2 keys, so a new key would make an older Fleet reject every record after a rollback. A running seat never changes generation; its next Start adopts the selection.
5. **Readers follow the pin.** Each row of the census above reads its "Must read" column, except the two rows marked as an opt-in precondition. The lease carries the generation id only, validated against the ledger per the trust boundary.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `fleet.harnessGenerations.root` (new leaf) | `ai/configBase.mjs` | absolute store root, `planeMember: false` | none | leaf JSDoc | AiConfig lints, `config-leaf-parity` |
| `fleet.harnessGenerations.{codexDesktop,antigravity,claudeDesktop}` (new leaves) | `ai/configBase.mjs` | per-family opt-in, default `false` | `false`: today's `harnessBinaries` path | leaf JSDoc | unit specs |
| Start for an opted-in family | `startAgentProvisioned.mjs` + `FleetLifecycleService.mjs` | bootstrap-admit the installed build if new, then launch the selected generation (Codex app + CLI from one generation) | named refusal with the latest rejection reason | JSDoc | unit specs + host witness |
| Lifecycle record, `status()` and lease harness generation | `FleetLifecycleService.mjs` | `{family, generationId, shortVersion, buildVersion}` on the record and in `status()`; the id alone on the lease | `null` in `status()` and no lease key for families that have not opted in | JSDoc | unit specs |
| `listHarnessGenerations({root, family})` (new engine export) | `harnessGeneration.mjs` | every admitted generation whose copy is present, newest first, its paths derived from the store root and its ledger entry; `selectHarnessGeneration` is its first entry | `[]` for a family with no ledger; `ledger-invalid` for a malformed one | JSDoc | `harnessGeneration.spec.mjs` |

**Decision Record impact:** aligned-with #1531, which amends `neomjs/neo-agent-institution#adr:0034-electron-shell-architecture`.

## Acceptance Criteria

- [ ] A family that has not opted in starts exactly as today, from the `harnessBinaries` path (unit).
- [ ] For an opted-in family, the first Start admits a new installed build once. A second Start with the same installed build materialises nothing (unit).
- [ ] An opted-in Start launches the selected generation's main binary. For `codex-desktop`, the bundled CLI resolves inside the same generation (unit).
- [ ] With no admitted generation, an opted-in Start refuses by name with the family's latest rejection reason and never launches `/Applications` (unit).
- [ ] The lifecycle record and `status()` report the pinned generation, and a running seat is not re-pinned by a later admission (unit).
- [ ] After a Fleet restart, `adoptLeasedSeat` recognises a seat running from its pinned generation. A lease naming an id the ledger does not list as admitted is invalid (unit).
- [ ] `desktopProfileInUse` reports a seat running from any admitted generation of its family as in use (unit).
- [ ] The new leaves pass the AiConfig lints and `config-leaf-parity`.
- [ ] Host witness: a dev Fleet with `codexDesktop` opted in starts a Codex seat whose process runs from `<root>/codex-desktop/<id>/ChatGPT.app` (`ps` receipt). This needs a GPT seat with budget, so it is operator-timed.

## Out of Scope

- Retiring unpinned generations (its own leaf; until it lands, the store keeps every admitted generation).
- Fetching (#1537, #1539).
- The Claude cutover: its probe and its managed-preferences policy.
- The cockpit view.
- Flipping any opt-in default.
- Making the two Codex app-server CLI readers (the wake daemon and the resume script) follow the pin. That is a precondition of opting `codexDesktop` in.

## Avoided Traps

- Resolving the generation inside the synchronous `start()` or on every `getHarnessBinaryPath` call. Admission is async and expensive, so it runs once in the provisioning path, and `start()` keeps binding the snapshot it is given.
- Falling back to `/Applications` when no generation is admitted. That copy is exactly the build admission refused.

## Related

Parent #1531 · #1534 · #1537 · #1539 · #1542 · #1530

Live latest-open + closed sweep: latest 20 open plus a search for `harness generation seat start pin launch admitted` at 2026-11-09T19:52Z; no equivalent found. #1531's subs are #1530, #1532, #1537, #1539 and #1542, none of them seat-side. A2A claims: Grace #1536, Maryam #1542, Julia #1539; no overlap. Own assignment: #1531 (parent).

Intake, 2026-11-10 (Ada):
- **Drift since `createdAt`:** `harnessGeneration.mjs`, `localWakeAdapters.mjs`, `prepareManagedAgentWorkspace.mjs`, `antigravityUpdate.mjs` and `codexDesktopUpdate.mjs` moved. Re-read: the engine API is unchanged apart from `readHarnessGenerationLedger`; wakes address the seat's pid; #1544's per-seat `autoCheckForUpdates` suppression does not depend on the bundle path.
- **Prescription checked:** `ai/services/fleet/FleetLifecycleService.mjs#assertRemoteMcpCapability` owns the concern. It makes the binary snapshot that `start()` binds.
- **Epic review:** #1531 is self-authored.
- **ADR 0019:** read. The new leaves follow §5: declarative `leaf()` declarations read at their use sites, the store root handed to the engine as a pure operation's scalar, and each new path registered in `config-leaf-parity.json`.
- **ADR successor-risk:** adr-aligned. Artifact #1543 (2026-11-09); ADR neomjs/neo-agent-institution#adr:0034-electron-shell-architecture (Accepted 2026-07-10); evidence §2.5 *Runtime generations for managed seats*. Items 1–3 are the pattern this leaf mirrors for vendor bundles. Item 6 puts a vendor-harness update outside the surviving-seat promise, so pinning seats to admitted copies contradicts nothing. Route: continue. The §2.5 amendment recording vendor-harness generations stays #1531's unfiled leaf.
- **Verdict:** valid-as-written, with the census sharpened.
- **Amended during implementation, 2026-11-10:** the pin moved from the seat runtime record to the lifecycle record (Fix 4, the rollback hazard), the census gained the two Start paths that pass no proof, the bootstrap runs before the catalog reads the harness (Fix 2), and the Contract Ledger gained the engine's `listHarnessGenerations`.

Origin Session ID: 1ef4ba01-d8ab-4b0e-bb7c-1ee8426dc5fe

Retrieval Hint: `query_raw_memories("seat Start pins harness generation launchBinaryPath assertRemoteMcpCapability codex CLI same generation")`

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-09T19:51:07Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-09T19:51:08Z @neo-opus-ada added the `enhancement` label
- 2026-11-09T19:51:08Z @neo-opus-ada added the `ai` label
- 2026-11-09T19:51:09Z @neo-opus-ada added the `architecture` label
- 2026-11-09T19:51:09Z @neo-opus-ada added the `agent-os` label
- 2026-11-09T19:51:12Z @neo-opus-ada added parent issue #1531
- 2026-11-09T20:12:34Z @neo-opus-ada cross-referenced by PR #1544
- 2026-11-09T20:20:57Z @neo-opus-grace cross-referenced by PR #1545
### @neo-opus-ada - 2026-11-09T20:27:26Z

**Handover (session sunset, 2026-11-09 20:15Z).** Owner: still Ada, next session. Code status: not started; no branch exists.

**Pickup protocol:**
1. Branch `ada/1543-seat-generation-pin` from `dev` (`a2b2afab` or newer). #1534, #1538 and #1541 are on `dev`; #1544 (Antigravity) is in review.
2. Start at `FleetLifecycleService.assertRemoteMcpCapability`. It already returns `{binaryPath, launchBinaryPath}`: for `codex-desktop` the CLI and the app separately, for `antigravity` through `resolveAntigravityStartContext`. Select the generation there, behind the per-family opt-in leaf, and bootstrap-admit the installed build when the ledger has neither admitted nor rejected it. Use `readHarnessGenerationLedger` (on `dev` now) for that check; don't parse the ledger.
3. Then make the readers in this body's census table follow the pin. The two that break silently are `adoptLeasedSeat` and `desktopProfileInUse`. The lease carries only a generation id, validated against the ledger (the trust boundary above).
4. Don't flip any opt-in default. The Codex opt-in can follow once this lands: wakes already target the process (#1372), and #1542 (PR #1545) generalises that.

**Prior art since filing:**
- `codexDesktopUpdate` and `antigravityUpdate` hand candidates to `materializeHarnessGeneration`. Their per-process rejection memory keyed by `admitterId` means a Start-time bootstrap does not need its own retry guard.
- The installed `/Applications` builds admit today: Codex 26.1103 as `aba6eda7…` and Antigravity 2.29.1 as `6d2b1851…` (receipt on #1531).

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-09T20:27:32Z @neo-opus-ada cross-referenced by #1531
- 2026-11-09T23:04:49Z @neo-opus-ada cross-referenced by #1552
- 2026-11-10T00:22:03Z @neo-opus-ada referenced in commit `7d09cdc` - "feat(fleet): an opted-in Start proves its selected harness generation (#1543)

assertRemoteMcpCapability resolves the family's selected generation for an opted-in GUI family, admitting the installed build once per Fleet process when its version is new to the ledger, and proves the app and the bundled Codex CLI inside that one copy. The proof carries the pin, start() takes its CLI from the proof, and the record keeps the pin. A family without a generation refuses by name with the latest rejection and never launches the installed bundle."
- 2026-11-10T02:55:29Z @neo-opus-ada referenced in commit `0d13cc8` - "feat(fleet): seat readers follow the pinned harness generation (#1543)

- A Start that brings no capability proof (no repository, or no remote
  MCP) launches an opted-in family's selected generation too, or refuses;
  startAgentProvisioned admits a new installed build before the catalog
  or the capability proof reads the harness.
- The Antigravity context resolves the selected generation and carries
  its pin; validation compares the pin and binary with the selection.
- status() reports harnessGeneration; a later admission never re-pins a
  running seat.
- The lease carries only the generation id. Adoption derives the binaries
  from this Fleet's ledger entry for it, and an id the ledger has not
  admitted voids the lease. A lease without one is written as before.
- desktopProfileInUse counts the configured binary and every admitted
  generation of the family, opted in or not.
- The Codex catalog asks the CLI inside the selected generation.
- Engine: listHarnessGenerations; selectHarnessGeneration is its first
  entry."
- 2026-11-10T03:04:30Z @neo-opus-ada referenced in commit `9d9f56c` - "chore(fleet): merge dev into the seat generation pin branch (#1543)"
- 2026-11-10T03:04:31Z @neo-opus-ada referenced in commit `3e49fc9` - "test(fleet): composed-start stub resolves no generation (#1543)

startAgentProvisioned now asks its lifecycle service for the family's harness generation before anything reads the harness; the composed-start spec's stub lacked the method, and the lifecycle contract in the JSDoc now names it."
- 2026-11-10T03:05:55Z @neo-opus-ada referenced in commit `7748194` - "test(fleet): the engine lists every admitted generation (#1543)"
- 2026-11-10T03:07:04Z @neo-opus-ada cross-referenced by PR #1571
- 2026-11-10T03:54:49Z @neo-opus-ada referenced in commit `57414f0` - "fix(fleet): a thrown admission is not kept for the process (#1543)

materializeHarnessGeneration refuses a build by resolving {admitted: false}, which stays tried for the Fleet process. It can also throw (an unreadable ledger, a full disk); that rejection was cached the same way, so every later Start of the build failed until the Fleet restarted. A thrown admission is now dropped. From Maryam's review 5836504522."
- 2026-11-10T04:04:59Z @tobiu referenced in commit `a3d45e9` - "feat(fleet): seats start from admitted harness generations (#1543) (#1571)

* feat(fleet): harnessGenerations leaves, per-family opt-in off by default (#1543)

The store root and one boolean opt-in per GUI family. Nothing reads them yet; the Start path that selects and pins a family's admitted generation follows on this branch.

* feat(fleet): an opted-in Start proves its selected harness generation (#1543)

assertRemoteMcpCapability resolves the family's selected generation for an opted-in GUI family, admitting the installed build once per Fleet process when its version is new to the ledger, and proves the app and the bundled Codex CLI inside that one copy. The proof carries the pin, start() takes its CLI from the proof, and the record keeps the pin. A family without a generation refuses by name with the latest rejection and never launches the installed bundle.

* feat(fleet): seat readers follow the pinned harness generation (#1543)

- A Start that brings no capability proof (no repository, or no remote
  MCP) launches an opted-in family's selected generation too, or refuses;
  startAgentProvisioned admits a new installed build before the catalog
  or the capability proof reads the harness.
- The Antigravity context resolves the selected generation and carries
  its pin; validation compares the pin and binary with the selection.
- status() reports harnessGeneration; a later admission never re-pins a
  running seat.
- The lease carries only the generation id. Adoption derives the binaries
  from this Fleet's ledger entry for it, and an id the ledger has not
  admitted voids the lease. A lease without one is written as before.
- desktopProfileInUse counts the configured binary and every admitted
  generation of the family, opted in or not.
- The Codex catalog asks the CLI inside the selected generation.
- Engine: listHarnessGenerations; selectHarnessGeneration is its first
  entry.

* test(fleet): composed-start stub resolves no generation (#1543)

startAgentProvisioned now asks its lifecycle service for the family's harness generation before anything reads the harness; the composed-start spec's stub lacked the method, and the lifecycle contract in the JSDoc now names it.

* test(fleet): the engine lists every admitted generation (#1543)

* fix(fleet): a thrown admission is not kept for the process (#1543)

materializeHarnessGeneration refuses a build by resolving {admitted: false}, which stays tried for the Fleet process. It can also throw (an unreadable ledger, a full disk); that rejection was cached the same way, so every later Start of the build failed until the Fleet restarted. A thrown admission is now dropped. From Maryam's review 5836504522."
- 2026-11-10T04:05:00Z @tobiu closed this issue

