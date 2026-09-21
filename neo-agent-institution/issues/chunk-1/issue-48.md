---
id: 48
title: Extract the cockpit source-read family into one fenced-read discipline
state: CLOSED
labels:
  - ai
  - refactoring
assignees:
  - neo-fable-clio
createdAt: '2026-08-29T11:23:18Z'
updatedAt: '2026-08-29T12:51:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/48'
author: neo-fable-clio
commentsCount: 0
parentIssue: 22
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-29T12:51:08Z'
---
# Extract the cockpit source-read family into one fenced-read discipline

## Context

First extraction cut under Epic #22 (operator verdict: FleetCockpit at 3,640 LOC is severe architectural debt; the epic's shape: pull responsibility families out along existing seams into `view/fleet` leaf siblings, behavior-frozen, one cut per PR). The cockpit landed at this size one uniform pattern at a time — nowhere more visibly than the source-read family this sub extracts.

## The Problem

Seven cockpit bridge-read verbs repeat ONE discipline per source, hand-rolled each time (~450 LOC of the class): resolve `globalThis.AgentOS?.fleet?.registryBridge` → verb-presence check → try/await → typed unavailable fallback envelope → read-generation fence (`++me.<x>ReadGeneration`, compare-before-write) → owner-held snapshot field → WRITE-time pane resolution (phase-blind accessor). The members: `loadCatchUp`, `loadMemories` (+ pre-await owner-held target), `loadSessionMemories` (+ pre-await drill hold, wire-param strip), `clearSessionMemoriesDrill` (terminal generation bump), `loadWakeRoutes`, `loadTasks` (+ in-flight accounting in `finally`), `loadOperatorInbox` (pane+subject gate, keep-last-truth on error), `loadOperatorIdentity` (no fence; record + posture + live pane set) with `deriveOperatorIdentityPosture`. Every new source re-types the discipline; drift between copies is the known failure mode the epic names.

## The Fix (behavior-frozen)

1. New leaf sibling `apps/agentos/view/fleet/cockpit/sourceReads.mjs`: one generic `readFencedSource(owner, descriptor, params)` core carrying the shared laws (verb check, typed fallback, fence, owner write, WRITE-time pane resolve) with the measured variance as explicit descriptor hooks — `preAwait(owner, params)` (memories target / drill hold), `wireParams(params)` (title strip), `inFlightField` (tasks accounting, released in `finally` on the read's OWN settle), `gate(owner, params)` + keep-last-truth error mode (operator inbox), `apply(owner, snapshot)` (operator identity's record/posture/pane set). A `SOURCE_READS` descriptor table names each source's contract in one place.
2. The cockpit keeps thin delegates with UNCHANGED signatures and semantics (`loadMemories(params) { return readFencedSource(this, SOURCE_READS.memories, params) }`); full behavioral docblocks move WITH the substance into the module; delegates carry @summary + @see.
3. Behavior frozen: zero semantic change; the FULL agentos unit tree green before and after; the existing owner-seam specs (memories ownerSeam.spec among them) stay green against the delegates; a focused `sourceReads.spec.mjs` pins the generic laws once (fence, fallback typing, write-time resolve, in-flight release-on-own-settle, gate short-circuit) instead of per-source stubs.

## Out of Scope (later subs of #22)

- The routing-matrix read class (`loadRoster` / `loadActivity` / `loadBrainHealth` — state-writing routing matrices, `reconcileRoster` coupling).
- The WRITE verbs (`markCatchUp`, `composeOperatorMessage` fan-out + re-poll) — the source-WRITE family.
- The vessel/tear-out and chrome-sync families; the file-size guard sub.
- Any behavior change whatsoever.

## Acceptance Criteria

- [ ] `sourceReads.mjs` exports the generic core + descriptor table; the seven read verbs delegate with unchanged public signatures.
- [ ] FleetCockpit `Container.mjs` shrinks by the extracted family (target ≥ 250 net lines out of the class).
- [ ] Full agentos unit tree green before and after (identical pass set — behavior frozen); existing owner-seam specs untouched and green.
- [ ] Focused module spec pins: generation fence (older read never writes), typed fallback on unwired AND throwing bridge, WRITE-time pane resolution (destroyed-pane-during-await case), tasks in-flight release on own settle, operator-inbox gate + keep-last-truth error mode.
- [ ] No unrelated visual baseline refreshed (AC-7 restamp per commit).

## Related

Parent: #22 (linked as sub) · pattern precedent: the `view/fleet` leaf siblings the epic cites · consumers proven tonight: the mailbox/memories drains (#41/#45) run through exactly these verbs.

Live latest-open sweep: latest 10 open issues re-checked at creation time — no equivalent. A2A in-flight: current claims cover Brain #71/#198/#215/#229/#231 (Emmy), Brain #64/#200 (Vega), Engine #17836-#17844 (Euclid/Mnemo/Ada) — zero overlap with the cockpit read seam.

Origin Session ID: 41859592-b7ee-4bce-bee3-f25644d9003b

Retrieval Hint: `query_raw_memories("cockpit source read family extraction fenced descriptor")`

Authored by Clio (Fable 5, Claude Code). Session 41859592-b7ee-4bce-bee3-f25644d9003b.


## Timeline

- 2026-08-29T11:23:20Z @neo-fable-clio added the `ai` label
- 2026-08-29T11:23:20Z @neo-fable-clio added the `refactoring` label
- 2026-08-29T11:23:26Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-08-29T11:29:53Z @neo-fable-clio cross-referenced by PR #49
- 2026-08-29T12:22:11Z @tobiu referenced in commit `ddd4813` - "fix(agentos): dispatch posture through the owner's virtual seam (#48)

Euclid's RA-1 on PR #49: loadOperatorIdentity called the module-private
deriveOperatorIdentityPosture directly, bypassing the cockpit's virtual
method — base dev dispatched me.deriveOperatorIdentityPosture(nodeId),
so an owner-side override stopped receiving the call (reviewer sentinel
control proved static-bypass at b7ad375). The module now dispatches
through owner.deriveOperatorIdentityPosture(nodeId) — the delegate
routes the default back into the module, overrides seat again — and a
focused sentinel witness pins the seam (red against the prior head).

RA-2 alongside: the honest extraction measure is Container.mjs
3,640 → 3,339 (−301, +34/−335); the earlier −295 was a pre-reshape
intermediate. 732/0 unit."
- 2026-08-29T12:40:07Z @tobiu referenced in commit `8191e20` - "refactor(agentos): extract the cockpit source-read family into one fenced discipline (#48)

First extraction cut of Epic #22 (FleetCockpit 3,640 LOC, operator-named
severe debt). Seven bridge-read verbs hand-rolled ONE contract per
source; the contract now lives once in AgentOS.util.CockpitSourceReads
(a static core.Base util per the view/util topology laws — the view tree
carries class modules only):

- readFencedSource(owner, descriptor, params) carries the shared laws:
  verb-presence check, typed unavailable fallback, read-generation fence
  (bump first — gate-refused intents still invalidate older in-flight
  reads, exactly the shipped order), owner-held snapshot write, and
  WRITE-time pane resolution through the phase-blind accessors.
- The measured per-source variance became explicit descriptor hooks:
  preAwait owner holds (memories target, drill session), wireParams
  (title strip, owner-derived inbox subject), inFlightField (tasks
  accounting released on the read's OWN settle), gate + errorMode 'keep'
  (operator inbox: honest unobserved refusal, last truth survives a
  throwing bridge).
- loadOperatorIdentity + deriveOperatorIdentityPosture move as named
  statics (deliberately outside the template: no fence, absence IS the
  state, a bridge throw propagates); clearSessionMemoriesDrill keeps its
  terminal-fence semantics.
- The cockpit keeps thin delegates with unchanged public signatures;
  full behavioral docblocks moved WITH the substance. Container.mjs
  3,640 -> 3,339 (-301, +34/-335).

Behavior frozen and witnessed: the full agentos tree passes with the
IDENTICAL set before/after (723) plus 8 new law witnesses pinning the
discipline once (fence loser writes nothing, typed fallbacks both ways,
destroyed-during-await never written, in-flight release on own settle,
gate short-circuit + keep-last-truth, terminal drill close, wire-title
strip) — 731/0; component battery 3/3; visual 6/6 untouched goldens."
- 2026-08-29T12:40:08Z @tobiu referenced in commit `a1fdd83` - "fix(agentos): dispatch posture through the owner's virtual seam (#48)

Euclid's RA-1 on PR #49: loadOperatorIdentity called the module-private
deriveOperatorIdentityPosture directly, bypassing the cockpit's virtual
method — base dev dispatched me.deriveOperatorIdentityPosture(nodeId),
so an owner-side override stopped receiving the call (reviewer sentinel
control proved static-bypass at b7ad375). The module now dispatches
through owner.deriveOperatorIdentityPosture(nodeId) — the delegate
routes the default back into the module, overrides seat again — and a
focused sentinel witness pins the seam (red against the prior head).

RA-2 alongside: the honest extraction measure is Container.mjs
3,640 → 3,339 (−301, +34/−335); the earlier −295 was a pre-reshape
intermediate. 732/0 unit."
- 2026-08-29T12:51:08Z @tobiu referenced in commit `c43f9b5` - "Merge pull request #49 from neomjs/agent/48-source-reads

refactor(agentos): extract the cockpit source-read family into one fenced discipline (#48)"
- 2026-08-29T12:51:08Z @tobiu closed this issue
- 2026-08-29T12:56:10Z @neo-fable-clio cross-referenced by #50
- 2026-08-29T13:00:10Z @neo-fable-clio cross-referenced by PR #51
- 2026-08-29T13:23:25Z @tobiu referenced in commit `1d3b244` - "refactor(agentos): home the cockpit read families on the view CONTROLLER (#50)

Cut 2 of Epic #22 — redirected mid-lane by the operator's architecture
ruling: passing 'owner' into a util file and triggering logic on it is
a functional mixin, not a responsibility move; it keeps dodging view
controllers. The named home for this logic is FleetCockpitController.

- BOTH read families move onto the controller: the #48 fenced snapshot
  template (readFencedSource + SOURCE_READS, six pane feeds) AND this
  ticket's liveness class (loadActivity / loadRoster / loadBrainHealth
  as named methods — their variance is structural, not templatable),
  plus the loss edges degradeWiredSurface / clearDegradedReason and the
  file-local helpers (boundedRead, toSafeDegradedReason).
- AgentOS.util.CockpitSourceReads is DELETED — the util shape lasted
  one review cycle and is not public history to preserve.
- The controller reaches view state through this.component; every
  collaborator call stays owner-virtual (the #49 RA-1 law). View state
  remains view-held in this cut; state.Provider migration is the #42
  follow-up.
- The view keeps thin delegates with unchanged public signatures for
  its existing callers (startLiveness, boot, vessel paths).
- Specs migrated mechanically to the new address: prototype fakes now
  drive FleetCockpitController.prototype with {component: fake} — the
  fake shapes and every assertion are untouched; the law spec drives
  controller verbs directly.

Exact numbers at head: Container.mjs 3,339 -> 2,981 (-358); Controller
350 -> 1,186; the dev-side util (+414 via #49) is gone. Epic total:
FleetCockpit 3,640 -> 2,981 across two cuts. Behavior frozen: 735/0
unit (identical pre-cut set + the law witnesses), component 3/3,
visual 6/6 untouched goldens."
- 2026-08-29T22:13:03Z @tobiu cross-referenced by #55
- 2026-08-29T22:43:57Z @neo-fable-clio cross-referenced by #22
- 2026-09-04T18:29:58Z @neo-fable-clio cross-referenced by #64

