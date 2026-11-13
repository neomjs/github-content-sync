---
id: 1662
title: Codex Desktop seats run the newest admitted build
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-13T02:16:00Z'
updatedAt: '2026-11-13T16:18:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1662'
author: neo-opus-ada
commentsCount: 3
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
closedAt: '2026-11-13T14:26:55Z'
---
# Codex Desktop seats run the newest admitted build

## Context

The operator, 2026-11-13: *"FM was supposed to handle harness updates for claude, codex and antigravity. since updates arrive multiple times a week, we need a permanent solution to handle it."* #1658 covers Claude Desktop; this leaf covers Codex Desktop.

Measured on the operator's Mac, 2026-11-13 ~01:55Z:
- GPT seats run `/Applications/ChatGPT.app` 26.1103.60933 (build 25894). Their Start sets `CODEX_SPARKLE_ENABLED=false` (`deriveHarnessLaunchSpec.mjs`), so their own updater is off.
- OpenAI's appcast (`https://persistent.oaistatic.com/codex-app-prod/appcast.xml`, read through `dev`'s `parseCodexSparkleAppcast`) already lists 26.1106.51428, published 2026-11-11T02:34Z.
- The Fleet's Codex acquirer is wired (`HARNESS_ACQUIRERS['codex-desktop']`, #1537 and #1587), but `fleet.harnessGenerations.codexDesktop` defaults to `false` and the packaged FM sets no override, so the job skips Codex as `not-opted-in` and the store holds no `codex-desktop` generation.

Codex seats are frozen one release behind.

## The Problem

#1531 point 6 holds Codex's opt-in until the two Codex app-server CLI readers outside the Fleet follow the seat's pin: the wake daemon's `deliverViaCodexAppServer` (`ai/daemons/wake/daemon.mjs`) and the `codex-app-server` resume in `ai/scripts/lifecycle/resumeHarness.mjs` both spawn `AiConfig.fleet.harnessBinaries.codex`, the `/Applications` CLI, whatever build the seat runs. Once a seat runs a newer generation, those two drive its app-server with an older CLI. The Fleet arms its own seats with `a2a-webhook` routes (`armFleetSeatWake.mjs`), so only a hand-configured `codex-app-server` route reaches those readers, and nobody has counted them: wake routes are owner-scoped, so no single seat can list them all.

## The Architectural Reality

- An opted-in Codex Start launches the generation's `ChatGPT.app` and its bundled CLI from the same copy (`FleetLifecycleService#harnessGenerationLaunch`). `seatModelCatalog.mjs` already asks the CLI inside the selected generation.
- The seat's pin is in its lifecycle record and lease (`harnessGenerationId`).
- Since #1658, `FleetLifecycleService#harnessGenerationOwnership` is the one opt-in read; Codex has no host gate, so the leaf alone decides.

## The Fix

1. **Census** (done, comment 6841848531). No reader drives a Codex seat with the installed CLI: the live wake receiver holds no `codex-app-server` route (one would name its own `adapterConfig.codexBinary`), `resumeHarness.mjs`'s branch is fail-closed behind `RUN_LIVE_CODEX_APP_SERVER=1`, and the legacy `daemon.mjs` is not running. Inside the Fleet every Codex CLI reader already follows the pin (#1543). #1531 point 6's precondition therefore holds, and the readers stay as they are.
2. **Opt-in.** `fleet.harnessGenerations.codexDesktop` defaults to `true`, as #1590 did for Antigravity, and its JSDoc says why: every Codex seat's Start turns its own updater off (`CODEX_SPARKLE_ENABLED=false`).
3. **Specs.** A subprocess spec whose Codex app is a stand-in pins the leaf to `false`, so the stand-in is never taken for an admitted build.

## Acceptance Criteria

- [x] The census of active `codex-app-server` routes is recorded on this ticket (comment 6841848531).
- [ ] `codexDesktop` defaults to `true`, and an opted-in Codex Start launches the selected generation's `ChatGPT.app` with the bundled CLI from the same copy (unit, the existing #1543 arm on the new default).
- [ ] A subprocess spec whose Codex app is a stand-in pins the leaf, so a default never decides it.
- [ ] The leaves pass `lint-config-template-ssot` and `check-aiconfig-antipatterns`.
- [ ] **Post-merge** `[L4-deferred — operator handoff needed]` (an FM build carrying this): an acquisition pass admits ChatGPT.app 26.1106.51428 or later with `source` set to the appcast's enclosure URL; a restarted Codex seat runs from `~/.neo-ai/harnesses/codex-desktop/<id>/ChatGPT.app` with its bundled CLI from the same copy (`ps`). This is #1543's AC-9, operator-timed because it spends GPT budget.

## Out of Scope

- The shared archive pipeline (#1660).
- Retiring old generations and the cockpit (#1531 point 7).

## Avoided Traps

- **The selected generation instead of the pin.** A seat keeps the build it started with until its next Start, so a reader that follows the family's newest build talks to a running seat with a newer CLI than it runs.
- **Opting in without the census.** A reader that drives a pinned seat with the installed CLI fails only when a wake does; the census is what rules it out, and a `codex-app-server` route added later names its own CLI.

## Related

#1531 (parent) · #1658 (Claude, the one opt-in read) · #1543 (AC-9) · #1537 · #1587 · #1590 (Antigravity's opt-in, the precedent) · #609 (the wake daemon's delivery engine, which owns `deliverVia*`)

Decision Record impact: aligned-with `adr:0019-aiconfig-reactive-provider-ssot` (one leaf's default changes) and `neomjs/neo-agent-institution#adr:0034-electron-shell-architecture` §2.5.

Not a `good first issue`: it is owned, and its census and receipt need the GPT seats and the operator's Mac.

Live latest-open + closed sweep: latest open, plus a search for `codex opt-in OR codexDesktop OR "codex app-server" pin` across open and closed (Out of Scope read on #1543 and #1590) at 2026-11-13T02:18Z; no equivalent found. #609 owns the delivery engine's shape, not which CLI it spawns. Backlog band: 156 open (red), so this folds under #1531. A2A claim sweep: latest 30 messages, all read states; no claim on Codex harness updates. MC sweep: "Codex seats never update; Sparkle disabled per seat; codex app-server wake delivery uses the Applications CLI", 5 results; no prior decision on the readers. Prior art for the post-merge AC: a Codex release's minification once broke `probeCodexDesktopCapabilities` (#1530), so a new build can fail admission by design until the probe follows. Own-assignment sweep: #1531 (the parent), #571, #133 and #1508; none overlapping.

Origin Session ID: 776d12fd-202a-486b-88c0-41605d050556

Retrieval Hint: "Codex Desktop seats frozen with Sparkle off and codexDesktop opt-in off; app-server CLI readers follow the seat's pin"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-13T02:16:00Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-13T02:16:01Z @neo-opus-ada added the `enhancement` label
- 2026-11-13T02:16:02Z @neo-opus-ada added the `ai` label
- 2026-11-13T02:16:02Z @neo-opus-ada added the `agent-os` label
- 2026-11-13T02:16:10Z @neo-opus-ada added parent issue #1531
### @neo-opus-ada - 2026-11-13T02:17:48Z

**Census, AC-1** (2026-11-13 ~02:15Z, the operator's Mac). No Codex app-server reader drives a Codex seat today:

- **The live wake receiver** (`ai/daemons/wake/receiver.mjs`, launchd `com.neomjs.agent-os-wake`, manifest `~/Library/Application Support/Neo/AgentOS/wake/routes.json`) holds 13 routes: 11 `osascript` (the three GPT seats among them), 1 `opencode-server`, 1 `kimi-pull-bridge`, and **0 `codex-app-server`**. Only adapter and identity fields were read. A `codex-app-server` route there would name its own CLI (`adapterConfig.codexBinary`, `localWakeAdapters.mjs`), not `fleet.harnessBinaries.codex`.
- **`resumeHarness.mjs`'s `codex-app-server` branch** is fail-closed: `assertCodexAppServerAllowed()` throws unless `RUN_LIVE_CODEX_APP_SERVER=1` (an operator probe) or the test mock is set.
- **The legacy `ai/daemons/wake/daemon.mjs`** (`deliverViaCodexAppServer`) is not running; `ps` shows `receiver.mjs` and `hostEdge.mjs` only. Its delivery engine is #609's.
- **Inside the Fleet** every Codex CLI reader already follows the pin (#1543): the Start's capability proof and auth command (`FleetLifecycleService`) and `seatModelCatalog.mjs`.

So #1531 point 6's precondition holds by census: no reader can drive a generation-pinned seat with the installed CLI. The leaf narrows to the opt-in; the body is amended to match.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-13T02:21:41Z @neo-opus-ada cross-referenced by PR #1664
- 2026-11-13T02:30:11Z @neo-opus-ada cross-referenced by #1665
- 2026-11-13T02:44:06Z @neo-opus-ada cross-referenced by #1531
- 2026-11-13T05:43:55Z @neo-opus-ada referenced in commit `00e355d` - "test(fleet): the journey child admits into its own store (#1662)

The fleet child inherits the host's HOME, and its harness-generation root defaulted to ~/.neo-ai/harnesses. Before the Codex pin, an opted-in run bootstrapped the Codex stand-in there and left a bundle-metadata-invalid rejection in the live codex-desktop ledger (02:20:26Z). The child now gets NEO_FLEET_HARNESS_GENERATIONS_ROOT under its run root, so isolation no longer depends on every family being pinned."
### @neo-opus-ada - 2026-11-13T05:44:20Z

**The stale codex-desktop rejection (defect-note, 03:34Z) is settled: this ticket's own journey spec wrote it.**

- **Writer.** `seatJourney.spec.mjs` starts its fleet child with the host's `HOME` and, until now, no `NEO_FLEET_HARNESS_GENERATIONS_ROOT`, so the child's harness store was `~/.neo-ai/harnesses`. While I built #1664, before its Codex pin existed, an opted-in run bootstrapped the stand-in `Codex.app` (it has no `Info.plist`) into the live store. That run left the `bundle-metadata-invalid` row at 02:20:26Z (`source: installed`), 73 s before #1664 opened.
- **Fix.** `00e355d1` on #1664 gives the child its own root, so isolation no longer depends on every family being pinned (the spec pins Claude and Codex, but not Antigravity, which defaults on). Probe: with Codex opted in, the Start is refused (admission rejects the stand-in) in three runs, and the live ledger stays byte-identical (620 bytes, mtime 02:20:26Z).
- **The stale row stays.** It is the live codex-desktop ledger's only entry. `selectedHarnessGeneration` names `rejected.at(-1)` only while the family holds no admitted generation, and on this host the first opted-in Start bootstraps `/Applications/ChatGPT.app`, which has an `Info.plist`. I'm not editing the FM's store by hand.
- **Why the row's versions read as PlistBuddy text:** `readBundleInfo` keeps PlistBuddy's stdout even when it exits 1 ("File Doesn't Exist, Will Create: …"). The reason recorded is right; the version fields are not. Captured as its own defect-note, outside this ticket.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-13T13:40:02Z @neo-opus-ada referenced in commit `7c7b266` - "chore(fleet): merge the #1662 head that carries dev (#1665)"
- 2026-11-13T14:26:54Z @tobiu referenced in commit `d1ff148` - "feat(fleet): Codex Desktop seats run the newest admitted build (#1662) (#1664)

* feat(fleet): Claude Desktop seats run the newest admitted build (#1658)

Claude's own updater is off by policy, and the Fleet had no Claude source,
so every Claude seat stayed on the build it was installed with. Homebrew's
public cask record now points at Anthropic's versioned release zip; the
record only decides whether to download (the vendor's URL shape, its own
version and commit, the SHA-256 checked before extraction), and admission
still proves the bundle.

The download half every acquisition shares lives once in
acquireHarnessArchive.mjs: per-family serialization, the same-admitter
rejection memo, a capped streaming digest check, ditto extraction, the
single .app and admission.

fleet.harnessGenerations.claudeDesktop defaults to true, and the host's
updater policy decides which updater owns the family instead of refusing
a Start: with the policy, seats start from the selected generation and the
Fleet acquires; without it, seats start from the installed bundle, whose
own updater keeps it current. This amends #1584's refusal.

* feat(fleet): Codex Desktop seats run the newest admitted build (#1662)

Every Codex seat's Start keeps its own Sparkle updater off, and
fleet.harnessGenerations.codexDesktop was off, so nothing brought a new
Codex build. The census on the ticket found no app-server reader that
drives a Codex seat with the installed CLI, and every Codex CLI reader in
the Fleet already follows the seat's pin, so the leaf now defaults to true.

The seat journey pins the leaf off for its stand-in bundle, as it pins the
Codex binary, so a default never decides which app it starts.

* test(fleet): the journey child admits into its own store (#1662)

The fleet child inherits the host's HOME, and its harness-generation root defaulted to ~/.neo-ai/harnesses. Before the Codex pin, an opted-in run bootstrapped the Codex stand-in there and left a bundle-metadata-invalid rejection in the live codex-desktop ledger (02:20:26Z). The child now gets NEO_FLEET_HARNESS_GENERATIONS_ROOT under its run root, so isolation no longer depends on every family being pinned."
- 2026-11-13T14:26:56Z @tobiu closed this issue
- 2026-11-13T15:26:29Z @neo-opus-ada cross-referenced by PR #1666
- 2026-11-13T15:49:03Z @tobiu referenced in commit `7b3076c` - "feat(fleet): retire harness generations nothing holds (#1665) (#1666)

* feat(fleet): Claude Desktop seats run the newest admitted build (#1658)

Claude's own updater is off by policy, and the Fleet had no Claude source,
so every Claude seat stayed on the build it was installed with. Homebrew's
public cask record now points at Anthropic's versioned release zip; the
record only decides whether to download (the vendor's URL shape, its own
version and commit, the SHA-256 checked before extraction), and admission
still proves the bundle.

The download half every acquisition shares lives once in
acquireHarnessArchive.mjs: per-family serialization, the same-admitter
rejection memo, a capped streaming digest check, ditto extraction, the
single .app and admission.

fleet.harnessGenerations.claudeDesktop defaults to true, and the host's
updater policy decides which updater owns the family instead of refusing
a Start: with the policy, seats start from the selected generation and the
Fleet acquires; without it, seats start from the installed bundle, whose
own updater keeps it current. This amends #1584's refusal.

* feat(fleet): Codex Desktop seats run the newest admitted build (#1662)

Every Codex seat's Start keeps its own Sparkle updater off, and
fleet.harnessGenerations.codexDesktop was off, so nothing brought a new
Codex build. The census on the ticket found no app-server reader that
drives a Codex seat with the installed CLI, and every Codex CLI reader in
the Fleet already follows the seat's pin, so the leaf now defaults to true.

The seat journey pins the leaf off for its stand-in bundle, as it pins the
Codex binary, so a default never decides which app it starts.

* feat(fleet): retire harness generations nothing holds (#1665)

Every GUI family now acquires its own builds several times a week, and no
copy was ever removed. After each family's acquisition, the job now retires
every admitted generation except the selected one, its rollback, and the
ones a seat holds: every seat record's pin and every lease on disk. A lease
that cannot be read keeps everything and says which. A retired copy's ledger
entry stays with a retiredAt stamp, so the build is never selected or
fetched again, and admitting it again clears the stamp. Retirement runs on
the family's admission chain, so no admission lands between the check and
a removal.

* test(fleet): the journey child admits into its own store (#1662)

The fleet child inherits the host's HOME, and its harness-generation root defaulted to ~/.neo-ai/harnesses. Before the Codex pin, an opted-in run bootstrapped the Codex stand-in there and left a bundle-metadata-invalid rejection in the live codex-desktop ledger (02:20:26Z). The child now gets NEO_FLEET_HARNESS_GENERATIONS_ROOT under its run root, so isolation no longer depends on every family being pinned.

* fix(fleet): retirement reads holds when it removes, fails closed (#1665)

The store calls `held` on the family's admission chain, in the synchronous step that removes, so a pin published while a retirement waits keeps its copy. A pending Start (it selects its build before its record pins it), an unreadable lease, a lease that is not the seat's own, or a pin that is not a generation id keeps every copy and names the seat. `isHarnessGenerationId` is the one id-shape check, shared by the ledger and the lease read."
### @neo-opus-ada - 2026-11-13T16:18:40Z

**Post-merge receipt, first half of AC-5 (2026-11-13).** FM installed at Brain `7b3076c8` (16:14Z). Its first acquisition pass:

```
16:16:53.158Z [fleet] harness acquisition codex-desktop: checked admitted (candidate 26.1111.21308, selected 26.1111.21308)
```

The live store now holds `7bd78de312cf` (26.1111.21308), `source` `https://persistent.oaistatic.com/codex-app-prod/ChatGPT-darwin-arm64-26.1111.21308.zip`, with its copy present.

**Open:** a Codex seat restarted onto that copy, with its bundled CLI from the same copy (`ps`). It is operator-timed because it spends GPT budget, so the box stays open.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


