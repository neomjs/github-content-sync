---
id: 1253
title: 'The boot focus read: one short business focus block, first in the handoff, derived from the business dashboard'
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-fable-mnemosyne
createdAt: '2026-10-24T20:14:40Z'
updatedAt: '2026-10-26T21:58:34Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1253'
author: neo-fable-mnemosyne
commentsCount: 2
parentIssue: 123
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
milestone: 'Boot cost: small answers by default'
---
# The boot focus read: one short business focus block, first in the handoff, derived from the business dashboard

## Context

The operator, 2026-10-24: when peers start fresh sessions they should read a short CEO dashboard that aligns their technical goals with the business goals; short is crucial. The same evening showed why: the business repository's dashboard, the surface every seat was told to read first, is 28.5 KB (≈ 7k tokens) and grew by 4 KB in one hour; the whole team re-read it after one herd prompt, ten recoveries at once. Measured on one Claude seat, a full recovery reached 125k tokens before any work, 89k of them fixed preload. The graph has the node types for this since July (#14446 delivered `BUSINESS_GOAL`/`METRIC` under #123) and no populated goal node: two `search_nodes` sweeps tonight found the schema, the `business-goal` concept nodes and nothing else. #957 (Clio) owns the `## Focus (v1)` handoff section for v1 rows, views and reach; nothing owns the business half or the size.

## The Problem

A fresh session has no cheap, authoritative answer to "what does the business need from me this week". The long answer exists (the private dashboard, the decision record, the week plan: ~65 KB together) and is right for the lead; it is wrong as a boot read for ten seats, and it is private to seats with the business clone. The Sandman handoff is read by every seat and rendered in the Fleet Manager's Golden Path pane, yet it carries graph gaps and the computed Golden Path only (issues; the launch playbook first) and nothing about the money events, the week's outcomes or the Nov 30 rule. Without the block, the Golden Path keeps recommending better code to a company whose stated problem is invisibility.

## The Architectural Reality

- Producer: `ai/services/graph/GoldenPathSynthesizer.mjs` writes sections; the handoff's reader is `get_sandman_handoff` (`sandmanHandoffStore.mjs:27`); the Fleet source extracts sections by heading; the `/overview` skill already expects a Focus section (#957's AC-6).
- #957's design: `## Focus (v1)` with four blocks, inputs through a host-edge reader with an `unknown` shape per block, projection under #123's `publicFlag` rule. This ticket adds the **business block** and the **cap**; it does not rewrite #957.
- Schema: `businessSchema` (`BUSINESS_GOAL`/`METRIC` with `{claimClass, falsifyingQuery, windowSemantics, confoundDisclaimer, publicFlag}`), delivered by #14446; `publicFlag` travels with the node.
- Boundary (#123, D#14430): mechanism public; goals, targets, revenue and client specifics private. The handoff lives on the plane's private data root; the public corpus projection must never carry the block.
- The readers already planned: the recovery runbook's step that reads the operator's focus artifact (Clio's outcome-8 leaf of the Oct 26 week plan) and the AGENTS.md boot rule that today points seats at the dashboard file.
- Design authority: none challenged; the handoff format is additive (ADR 0033's boundary).

## The Fix

**Sequencing (the operator via Emmy, 2026-10-25 00:07 CEST): the dashboard is a session-start read before any context-recovery or tool integration, and the Gemini seats run under a 258k cap — so the first screen must carry the business-to-technical alignment and each seat's actionable index as a plain document FIRST (Emmy's document-only cut, this weekend); the node and the handoff section below consume that screen afterwards and add no dependency to it. Links alone fail a seat that cannot afford the trail.**

1. **One source, one writer (refined 2026-10-25 with Emmy, the Map-vs-Atlas pass on the business repository): the focus block IS the dashboard's first screen.** No second file. `strategy/DASHBOARD.md` becomes the map: the focus block of at most 1,200 bytes (≈ 300 tokens) at the top — the question of the day and its decision rule with the date, the money events' next dates (≤ 3 lines), the week's outcomes one line each with the owner, the dates of the week — followed only by trails, one line per record (the decision record, the week plan, the by-seat block, the review-pack plan, each area file with its State word), each trail naming the condition under which it is read. The whole map stays under 2,500 bytes. **The guard is a repository-workflow change, not a sentence (Emmy, 2026-10-25): the business repository has no package or check hook today; until AC-7 lands, the writer measures the bytes at each commit and reports them, and nothing here calls the cap enforced.** The cap is held by that guard, never by habit: the file doubled from 28.5 KB to 64.8 KB in three hours on 2026-10-24 under one careful writer, and the document-only cut brought it to 2,272 bytes at `13dd6c1` (business #9) the same night — the plain-document half of this ticket is delivered; what remains here is the node, the section and the guard. History (convergence notes, evening notes) goes to dated immutable log files, linked once. No client names, no prices (categories and dates only, per #123). The lead remains the one writer.
2. **The node.** The golden-path lane (the synthesizer, container-plane — design note 2026-10-26: the host edge is graphless), when `orchestrator.focusSourcePath` is configured (the section below), upserts the block as the first populated `BUSINESS_GOAL` node, id `business-goal-focus` (the schema's prefix), `publicFlag: false`, `windowSemantics` = the week, `falsifyingQuery` = the dashboard commit it was derived from. The graph now has the business in it; Golden Path v2 (#122) can rank against it later; nothing ranks on it here (ADR 0033: reporting, additive).
3. **The section.** The synthesizer renders the node as `## Business focus` **as the first section of the handoff**, before the gap audit, with the source commit and the as-of time; when the block is absent the section says `unknown` with the reason; **when the block is over the cap nothing is truncated (a live instruction cut mid-sentence or mid-UTF-8 is worse than none — Emmy): the previous in-cap node stays, flagged `stale: over-cap` with the byte count and the source commit, and the section renders that previous block with the flag.** With the sibling `get_sandman_handoff` filing, the section is one addressable slug (`focus`) and a bare read returns the route plus this block.
4. **The readers.** The recovery runbook reads the section first (Clio's leaf; this ticket hands it the surface); the AGENTS.md boot rule points at the section instead of the 28 KB file (a one-line neo-agent-skills follow-up, filed when this lands); `/overview` prints it; the Fleet pane renders it with #957's envelope field.

## The source is configured, not assumed (the operator via Emmy, 2026-10-25 00:09 CEST; ADR 0019 read in full before this section)

**Our team's need is not a shared prerequisite.** Other teams may run the Agent OS with no business dashboard at all; the capability may one day be a paid extension. So the shared recovery path and the shared tools consume a **configured, optional** focus source, and Neo's deployment enables its own private one:

- **One leaf, empty by default.** `orchestrator.focusSourcePath` — `leaf('', 'NEO_AI_FOCUS_SOURCE_PATH', 'string', {planeMember: false, planeMemberReason: 'an operator-placed private file outside the plane, never beneath its root'})` in the Tier-1 config base; **no default path, no required-ness** (the `authorityProfile` precedent: a declared role, never an inherited one — here a declared source, never an assumed one). Declaration and env binding belong to the leaf alone (ADR 0019 §2, §5.2); the business clone's location is a deployment input, never a literal in code.
- **Read at the use site, by the golden-path lane only (design note 2026-10-26: the host edge is graphless and owns no plane path).** The synthesizer, in the orchestrator container, reads `AiConfig.orchestrator.focusSourcePath` where it runs; empty → the reader does nothing and the synthesizer writes no section. No pass-along into the synthesizer's arguments (B5), no `?.` (B3), no second resolver (C1), no global seed of a `business-goal-focus` node, no requirement that any team create a dashboard.
- **Absence is ordinary.** An unconfigured deployment keeps today's recovery path and today's handoff; the section is absent, not `unknown`, and nothing logs an error. Only a configured path that cannot be read reports `unknown` with its reason.
- **Tests isolate by construction** (§5.4): the pure read-and-cap function takes the path as a scalar (rung iii); no spec mutates `AiConfig` (B4).
- **The block's content follows the equal-peer working model (the operator's correction via Emmy, 2026-10-25 00:25):** the human is an equal peer and the gardener (his words, 2026-10-25 00:26: "I am an equal peer too, willing to help and dive into topics when needed, plus gardener role"): shared business and technical judgment includes him, and his row reads "joins where needed"; the human-only execution boundaries — the merge gate, signatures, account actions — are boundaries, not his role, so the block never casts him as an external decision authority or as a hands-only executor; authority is shared by the peers, and the block says so where it names owners. Peers managing a company, not only its technology, is the story the block tells. This is Neo's configured focus, never a governance requirement the capability imposes on another team; the fixtures (AC-1) carry the same distinction.
- **Two things stay separate** from the generic small-entrypoint discipline of this ticket: the private business-goal projection (Neo's own, `publicFlag: false`) and any commercial extension built on this capability — the paid-extension idea is a product option recorded here, not a licence decision or an entitlement mechanism.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| the focus block = `strategy/DASHBOARD.md`'s first screen (business repo, private) | the lead (one writer); the map cap in the business repository's workflow | ≤ 1,200 bytes; the five content lines above; the whole map ≤ 2,500 bytes with trails only | over the cap → the previous in-cap node stays, flagged `stale: over-cap` with the byte count and source; nothing truncated | the file's own header | the cap asserted by the host-edge writer's unit arm |
| `orchestrator.focusSourcePath` (Tier-1 leaf, new) | ADR 0019 §2/§5 (declarative leaf, read at the use site); §10.5 (`planeMember: false` + reason) | empty default; env `NEO_AI_FOCUS_SOURCE_PATH`; read only by the golden-path lane (the synthesizer) | empty → no read, no section, no error (absence is ordinary) | the leaf's metadata + `learn/agentos/AiConfigModel.md` | the config-parity lint green; a unit arm: empty leaf → the producer emits no section |
| `BUSINESS_GOAL` node `business-goal-focus` | `businessSchema` (#14446), #123's boundary | upserted by the golden-path lane with `publicFlag: false` only when the source is configured | no file → no node write, the previous node stays with its as-of | schema docs | unit arm: the node validates; a negative projection test: the public corpus projection excludes it |
| `## Business focus` handoff section | the synthesizer (producer-owned) | first section; as-of + source commit; `unknown`/`over-cap` shapes | synthesizer runs without the node → the section says `unknown` | synthesizer section JSDoc | snapshot arm with and without the node |
| `get_sandman_handoff` slug `focus` | the sibling filing | returned with the default read | — | — | the sibling's arms |

Decision Record impact: aligned-with ADR 0033 (additive reporting); **depends-on ADR 0019 (one new declarative leaf, read at the use site, `planeMember: false`)**; depends-on #14446's schema; none amended.

## Acceptance Criteria

- [ ] AC-1 **Fixture: business `strategy/DASHBOARD.md` at `13dd6c1` (published 2026-10-25 00:27 CET, business #9, Emmy; operator-approved): the whole map 2,272 bytes, the first screen through the by-seat rows 1,196 bytes, the original retained byte-for-byte as `strategy/2026-10-24-dashboard-record.md`; the map carries its own rule — replace current rows, archive history, measure the first screen ≤ 1,200 and the map ≤ 2,500.** The dashboard's first-screen block of ≤ 1,200 bytes becomes a `BUSINESS_GOAL` node `business-goal-focus` with `publicFlag: false`; a 1,300-byte block leaves the previous node in place, flagged `stale: over-cap` with the byte count and the source commit, and truncates nothing (unit arms).
- [ ] AC-2 The handoff's first level-two section is `## Business focus` carrying the block, the as-of time and the source commit; absent node → the section reads `unknown` with the reason (snapshot arm).
- [ ] AC-3 The negative projection test: the public corpus projection never contains the section or the node.
- [ ] AC-4 The section costs under 400 tokens in a bare `get_sandman_handoff` read (byte count asserted).
- [ ] AC-8 (the optional source) With `orchestrator.focusSourcePath` empty, a synthesizer run writes no focus section and no error, and a seat's recovery is byte-identical to today's; with it set, the section appears. The leaf carries `planeMember: false` with its reason; `check-aiconfig-antipatterns`, `lint-config-template-ssot` and the config-parity lint stay green; no spec mutates `AiConfig`.
- [ ] AC-7 (the guard) A commit guard in the business repository's workflow fails a `DASHBOARD.md` over 2,500 bytes and a first-screen block over 1,200 bytes; the acceptance names the hook, its run and one refused fixture commit. Until it lands, every map commit message carries the measured bytes.
- [ ] AC-5 (post-merge) one fresh session of another family reads the section first and names its week row from it without opening the business dashboard (receipt on this ticket); the runbook and boot-rule follow-ups are filed with their numbers here.
- [ ] AC-9 **(the source, decided — the operator 2026-10-26 22:5x CET: a tenant-repo mirror)** `orchestrator.focusSourcePath` points at the business repository's mirror under `tenantRepoMirrorRoot`; the repository is added to `tenantRepos` **only after #1309 (the release artifact exports the default tenant only, with a row-level guard) is on `dev`** — an engine release must never carry a tenant row; the acceptance names the #1309 merge commit and one scoped release receipt.

## Out of Scope

#957's v1/views/reach blocks and its host-edge GitHub reader; any ranking change (#122); the Fleet pane leaf (Institution #312); metrics ingestion (#123 Leaf 3); the public projection of any business number.

## Avoided Traps

- A hardcoded business-clone path or a globally seeded `business-goal-focus` goal: our dashboard is our deployment's input; another team's plane has none and must not know ours.
- Treating absence as a failure: an unconfigured source is the ordinary case for every deployment but ours.

- Putting the block on the context frontier: measured at 160 KB tonight, it cannot be a boot surface.
- A second writer: the block is derived from the dashboard's top block by the lead; peers send state lines, never edits.
- A cap enforced by prose: the writer measures bytes and records `over-cap`; a seat never reads a truncated block without knowing it.
- The dashboard itself as the node body: 28 KB of record is the long form, linked once.

## Related

Parent #123 (the business engine; this is the first populated goal node); the Map-vs-Atlas pass on the business repository and Stage (Emmy, 2026-10-25: the dashboard as the map, #363 the Stage README front door, #362/#364 untouched); sibling #957 (the Focus (v1) section — same handoff, different blocks; the cap and the ordering are proposed to its author by comment tonight); #14446 (the schema, done); #122 (ranking later); the sibling filings on the boot cost tonight (handoff default, frontier default, mailbox projection, skills description budget); the Oct 26 week plan's outcome 8 (Clio's runbook leaf); the business repository's dashboard READ THIS FIRST block (private).

Live latest-open sweep: the latest 20 open issues, created-descending, at 2026-10-24 19:5xZ; no equivalent (#957 is the closest and is linked, not duplicated). A2A in-flight claim sweep: the latest 30 `[lane-claim]` rows, none on this scope. MC sweep: three problem-noun queries (handoff payload, frontier payload, boot cost), 6 results each, initialization noise, no prior decision found; the June concept and D#14430 are the record and are cited. Own-assignment sweep: 11 open, none overlapping (#122 and #117 are mine and are the ranking layer this ticket stays out of). Structure map: `ai/services/graph` (the synthesizer and its section modules), `ai/services/memory-core` (the graph writers); the host-edge writer follows #957's reader pattern — structural pre-flight runs at authoring.

Origin Session ID: 542d408d-b0eb-497c-8f57-a19ca5833f5a

Retrieval Hint: "business focus block handoff first section BUSINESS_GOAL node publicFlag FOCUS.md 1200 bytes boot read"

## Timeline

- 2026-10-24T20:14:40Z @neo-fable-mnemosyne assigned to @neo-fable-mnemosyne
- 2026-10-24T20:14:41Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-10-24T20:14:41Z @neo-fable-mnemosyne added the `ai` label
- 2026-10-24T20:14:42Z @neo-fable-mnemosyne added the `architecture` label
- 2026-10-24T20:14:42Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-10-24T20:16:14Z @neo-fable-mnemosyne added parent issue #123
- 2026-10-24T20:17:12Z @neo-fable-mnemosyne cross-referenced by #957
- 2026-10-24T20:59:53Z @neo-fable-mnemosyne added this to the **Boot cost: small answers by default** milestone
- 2026-10-24T21:49:04Z @neo-fable-mnemosyne cross-referenced by #1258
- 2026-10-25T15:01:50Z @neo-gemini-julia cross-referenced by PR #1272
### @neo-fable-mnemosyne - 2026-10-26T21:39:03Z

## Design note for the implementer (Mon 2026-10-26, Mnemosyne) — the reader is the golden-path lane, not the host edge

Read against `origin/dev` at 4b5577d9, ADR 0019 in full first (gate 10). One premise of the body does not survive the code, and one heading would have cost the boot read what it was meant to save. Both are corrected in the body tonight; the reasoning is here.

### 1. The host edge cannot write the node

`ai/daemons/orchestrator/daemon.mjs` (`requiresOrchestratorPlane`, and the boot sequence): *"Host-edge roles must remain graphless"* — the role *"owns no Docker plane path"*. The Compose plane (`deploy/cloud/docker-compose.yml`) gives every container a **private** plane root (`orchestrator-plane-root`, `mc-server-plane-root`, `kb-server-plane-root`) and shares only channel volumes: sqlite, the handoff (orchestrator-write, mc-server `:ro`), deployment-state, rem-runs, vector-generation, heap-observation. A host process reaches the graph only through the MCP bridge the seats use. So §Fix 2's "host-edge reader upserts the node" is retired — it had no channel.

### 2. The reader is the synthesizer, in the orchestrator container

`GoldenPathSynthesizer.synthesizeGoldenPath` runs as the `golden-path` lane (`scheduling/pipeline.mjs:540`, container-plane authority) and writes `sandman_handoff.md` into `shared-handoff-data`. It reads `AiConfig.orchestrator.focusSourcePath` **at its own use site** (a new `import AiConfig from '../../config.mjs'`; today it only reads `Memory_Config`) and hands the string to a pure reader (§5.4 rung iii). The path must be readable **inside the orchestrator container**; two sources qualify, and which one is a deployment input the operator sets:

- **(a) a tenant-repo mirror.** `tenant-repo-sync` is live on the plane (enabled, 5 repos from the yaml tier, one tenant, last success 21:33Z tonight). Mirrors land under the orchestrator's private root at `<orchestrator.tenantRepoMirrorRoot>/tenant-repos/<tenant>/<repo>` (`deriveTenantRepoMirrorPath`, `tenantRepoAccessContract.mjs:567`) and refresh on the 30-minute cadence. **The business repository is not among the five today** — its `origin/main` and its recent history match none of the five `lastIngestedRev`s (two of them are `create-app` and `github-content-sync`; identities are hashed in the deployment state, by design). Adding it makes the source `…/tenant-repos/<tenant>/business/strategy/DASHBOARD.md`, the source commit = the mirror's HEAD, freshness = the sync. Side effect, stated: the strategy repository enters the private tenant corpus (`ask_knowledge_base` over it — a feature; a visibility obligation on the tenant seam that already exists for the client repos).
- **(b) a read-only mount.** The business clone (or a copy of the one file) bind-mounted into the orchestrator service by a Compose `volumes:` line, the container path in the leaf. No sync; freshness = the clone on the host; the source commit = `git rev-parse HEAD` of the mount when it is a clone, else the file's sha256 + mtime.

My recommendation is (a): one mechanism that already runs, the commit identity comes free, and it is the capability's own shape for another team (their dashboard is a file in their tenant repo too).

The mc-server also triggers the synthesizer after consolidation (`MemoryService.mjs:3315`); there the path is unreadable and the handoff mount is `:ro`. The reader therefore degrades to `unknown` + reason and never throws; the write it cannot do there is pre-existing behaviour.

### 3. The leaf

`ai/configBase.mjs`, in `orchestrator` beside `tenantRepoMirrorRoot` (~line 1497): `focusSourcePath: leaf('', 'NEO_AI_FOCUS_SOURCE_PATH', 'string', {planeMember: false, planeMemberReason: 'an operator-named file the orchestrator container can read — a tenant-repo mirror or a mounted file; empty means no focus source, and a plane-anchored default would invent one'})`. `leaf(defaultValue, env, type, metadata)` per `ConfigProvider.mjs:82`. One reader (the synthesizer); no pass-along (B5), no `?.` (B3), no second resolver (C1); `check-aiconfig-antipatterns`, `lint-config-template-ssot`, the config-parity lint stay green.

### 4. The node

`GraphService.upsertNode` (the synthesizer already calls it for gap pruning). **Id from the schema:** `createBusinessGoalId('focus')` → `business-goal-focus` (`ai/graph/businessSchema.mjs`; the body's `business-focus` yields to the prefix). Properties: the five `REQUIRED_BUSINESS_PROPERTIES` — `claimClass: 'derived'` (the validator admits only the metric classes), `falsifyingQuery: 'git show <sourceCommit>:strategy/DASHBOARD.md — the first screen'`, `windowSemantics: 'week:iso'`, `confoundDisclaimer: 'the lead's reading of the week, not a measurement'`, `publicFlag: false` — plus `lifecycle: 'active'`, `block`, `bytes`, `sourcePath`, `sourceCommit`, `asOf`. `validateBusinessProperties` before the upsert. Over the cap: no upsert, the previous node stays, the section renders the previous block flagged `stale: over-cap (<bytes> B, <commit>)`.

### 5. The section and its slug — the heading changes

`## Focus (business)` would have collided: `sandmanHandoffSections.headingToSectionSlug` strips a trailing parenthetical, so it maps to `focus`, the same slug as #957's `## Focus (v1)`, and `splitSandmanHandoffSections` **concatenates** same-slug sections — a boot read asking for the business block would have paid for the v1 blocks too. The heading is **`## Business focus`** → slug `businessFocus` by the generic rule. Add it to `KNOWN_HANDOFF_SECTION_SLUGS` and to `DEFAULT_HANDOFF_SECTIONS` (`['businessFocus', 'goldenPath']`), so a bare `get_sandman_handoff` returns it first (AC-4; #1251's default extended by one slug). Placement: directly after the H1 line of the handoff, before the gaps intro sentence (which belongs to the gaps section and moves below it) — the first level-two heading in the file. Module: `ai/services/graph/businessFocusSection.mjs` beside `activePrCycleSection.mjs` (sibling-file fast path of the structural pre-flight): `readFocusBlock({path, capBytes})` pure, `buildFocusNode({block, sourceCommit, asOf})`, `renderBusinessFocusSection({status, node, previousNode, asOf})`; the synthesizer gets the static delegating wrapper the other section modules have.

### 6. The arms

Reader: empty path → `absent` (no section, no error); unreadable path → `unknown` + reason; the 1,196-byte fixture (`13dd6c1`'s first screen) → `ok`; a 1,300-byte block → `over-cap` with the byte count, nothing truncated. Node: validates; the previous node survives an over-cap pass. Renderer: three snapshots (ok, unknown, over-cap-with-previous). Projection: the existing negative-projection shape asserts the node and the section never reach the public corpus. Store: `businessFocus` in the default read; the section under 1,600 bytes in the bare read. No spec mutates `AiConfig`.

### 7. Applied to the body tonight

§Fix 2 and the "configured, not assumed" section: the reader is the golden-path lane; AC-1 and §Fix 2: the node id is `business-goal-focus`; §Fix 3, the ledger and AC-2: the heading is `## Business focus`. Everything else stands. The one open input is the source, (a) or (b) — the operator's; an Opus seat implements Wed–Thu per the week plan.

Mnemosyne · Claude Fable 5.1 · Claude Code

- 2026-10-26T21:57:56Z @neo-fable-mnemosyne cross-referenced by #1309
### @neo-fable-mnemosyne - 2026-10-26T21:58:34Z

**Source decided (the operator, Mon 2026-10-26 22:5x CET): (a) the tenant-repo mirror** — "if we want to get business repo files into the KB, a tenant-repo mirror is smarter" — with one hard rule: engine releases export the engine corpus only; no other tenant's items may be in that artifact. Reading the export path tonight showed it cannot guarantee that today (the release artifact is the whole shared collection, no tenant filter, name-only guard) → filed #1309; AC-9 added above: the business repository joins `tenantRepos` only after #1309 is on `dev`. The implementer's order is therefore #1309 first, then this ticket's leaf, node and section.

Mnemosyne · Claude Fable 5.1 · Claude Code

- 2026-10-26T22:24:12Z @neo-opus-grace cross-referenced by PR #1310

