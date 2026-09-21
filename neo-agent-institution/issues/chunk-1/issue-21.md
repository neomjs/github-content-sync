---
id: 21
title: 'System view: plane health + diagnostics for the connected instance'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-08-18T08:21:55Z'
updatedAt: '2026-09-05T15:18:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/21'
author: neo-fable-clio
commentsCount: 4
parentIssue: 10
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-05T15:18:06Z'
---
# System view: plane health + diagnostics for the connected instance

# System view: plane health + diagnostics for the connected instance

## Context

Operator, live session 2026-08-18: "diagnostics views (e.g. OC, MC, KB container logs)." The cockpit renders fleet truth; the PLANES that produce it — the containers/host processes of the connected Agent OS instance — have no surface at all.

## The Problem

When a plane degrades today, diagnosis runs through MCP log-proxies (REM-pipeline state, self-heal recent events, healthcheck collection counts, tenant-sync repo states — the pattern peer-verified 2026-08-13: no raw-log surface exists on any plane) or the operator drops to a terminal. A mission control that cannot show its own engine room sends the operator away at exactly the wrong moment. And subject-wise this is NOT fleet content: planes ≠ agents — mixing plane logs into the fleet's reading tabs would blur the plane separation the architecture holds strictly.

## The Architectural Reality

- The local instance's plane inventory (`ai/deploy/docker-compose.local-agent-os.yml`): chroma, kb-server, mc-server, orchestrator, fleet-server, ingress.
- Health/deployment truth exists today as MCP tools (MC/KB healthcheck, deployment-state snapshot, deployment inspection, REM-pipeline state) reading the orchestrator's snapshot file — and NOT on the fleet wire the cockpit speaks (`FLEET_WIRE_METHODS`, verified 2026-09-04): neomjs/neo-agent-brain#314 adds the `fleetDeploymentState` verb, a bounded, observe-only projection of that snapshot.
- Exposure-surface precedent: D#14501 (closed) — the R3-safe control-plane exposure for the boot-identity fact + restart actuator; neomjs/neo-agent-brain#47 (open) — probes must declare observe-vs-mutate; neomjs/neo-agent-brain#54 (open) — external-plane recovery.
- Keeper-view pattern: `apps/agentos/view/Viewport.mjs` shell tab items — System joins Home · Fleet · Accounts · Chat per neomjs/neo#17269's navigation model (a distinct SUBJECT earns a distinct place).

## The Fix

A **System** keeper view for the CONNECTED instance (name provisional — decided in neomjs/neo#17269's navigation-model review, not silently here):

1. **Per-plane cards** — state, version, uptime, deployment snapshot — binding `bridge.fleetDeploymentState()` (neomjs/neo-agent-brain#314) through the authenticated bridge — its `ok` / `stale` / `unavailable` states and the older-server `unsupportedMethod` closed state each render as a first-class, reason-carrying card state.
2. **Logs region per plane** — renders once the log-serving verb (sibling service ticket, filed together) lands; until then the honest not-yet-wired state naming that sibling. Never a fabricated stream.
3. **Observe-only by design** — the restart-actuator class (D#14501 lineage) stays out of this cut, and the view SAYS it is observe-only (#16856's declaration discipline).

## Contract Ledger Matrix (consumer side)

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| twin `FLEET_WIRE_METHODS` (existing, `apps/agentos/config/fleetWireMethods.mjs:15`) | neomjs/neo-agent-brain#314 (producer) | gains `'fleetDeploymentState'` so `installFleetBridge` exposes `bridge.fleetDeploymentState()` | none — additive | JSDoc | the bridge spec's method-set arm |
| the read owner — implemented as `apps/agentos/util/DeploymentStateRead`, run by the cockpit's liveness owner on its fences (boot · liveness tick · reconnect); no separate System controller | this ticket; PR #114 | one `fleetDeploymentState` envelope per read, generation-fenced, stamped with the answering bridge's `profileId` and the reader's `observedAt` | a transport failure keeps the last-known picture and publishes the typed observation on `systemConnection`; an older server's `unsupported-method` lands `unavailable` with that reason | JSDoc | `livenessLifecycle.spec` deployment-state arms, incl. the instance-switch control |
| Viewport provider leaves `deploymentState` (leaf-complete: `{state: ok\|stale\|unavailable\|null, reason, generatedAt, ageMs, profileId, observedAt, services[], maintenance}`) and `systemConnection` (`{state, reason}`) | this ticket; producer shape from neomjs/neo-agent-brain#323's projection | the System view binds both; a picture stamped by another instance's bridge never renders under the bound profile; a failed read qualifies the retained picture as last known and its age advances from `observedAt` | never answered → not observed; foreign → no picture from this instance yet | JSDoc on the Viewport and the view | `container.spec` provenance + lost-contact arms |
| plane cards (new) | §04 TOKENS roles; the neomjs/neo#16744 reason-carrying discipline | one card per `serviceKey`: status, disposition, classification, observedAt, restart churn, memory pressure, diagnosis headline | `stale` → the card says how old; `unavailable`/`unsupported` → one reason line for the whole view, no per-card guessing | none | visual golden + the design sketch review |
| logs region (existing AC) | neomjs/neo-agent-brain#27 | honest absence naming #27 until its verb exists | never a fabricated stream | none | the same golden |

## Acceptance Criteria

- [ ] Design sketch (§04-consistent) reviewed BEFORE implementation (spec-first gate).
- [ ] Plane cards bind `fleetDeploymentState` (neomjs/neo-agent-brain#314); absent/unreachable planes render reason-carrying states (the neomjs/neo#16744 vocabulary discipline applied plane-side), never a generic offline. *[L3-deferred — operator handoff needed: the live rendering against a rebuilt plane carrying Brain #325 (`dd2109f`) is #10's residual (its discharge condition is recorded there); the unit/visual half is delivered by PR #114.]*
- [ ] The view is observe-only and declares it; no mutation affordance in this cut.
- [ ] Logs region: honest absence + named sibling ticket until the verb exists; per-plane bounded log rendering once it does.
- [ ] Instance-scoped: the view always names WHICH instance it diagnoses (consumes the instance switcher's scope, sibling ticket filed together).

## Out of Scope

The log-serving verb (sibling service ticket) · restart/remediation actuation (D#14501 / neomjs/neo-agent-brain#54 lineage — a future, separately-gated cut) · alerting/notification.

## Related

Epic neomjs/neo-agent-institution#10 (parent) · neomjs/neo#17269 (keeper placement) · **Blocked by neomjs/neo-agent-brain#314** (the wire verb) · logs sibling: neomjs/neo-agent-brain#27 (bounded, redacted plane-log reads) · instance switcher: shipped (`Viewport` `instance-switcher`, the scope this view consumes) · D#14501 · neomjs/neo-agent-brain#47 · neomjs/neo-agent-brain#54 · neomjs/neo#17288.

Live latest-open sweep: latest 20 re-checked 2026-08-18T08:15Z, no equivalent; A2A herd window clean (one unrelated engine claim neomjs/neo#17327).

Intake 2026-09-04 (author; the earlier-session drift probe fired on 7 declared surfaces → full gate): premise corrected in this body (the verbs live on the MCP plane, not on the wire); parent epic-reviews by @neo-gpt (2026-07-06) and @neo-opus-grace (2026-07-10); classification `needs-relinking` → relinked here → `valid-as-written` on the narrowed cut (cards + honest logs absence; the design-sketch AC stays first).

Origin Session ID: ca3c67ac-a3d6-4e93-98e0-c5f7f65011ee

Retrieval Hint: `query_raw_memories("system view plane health diagnostics logs keeper view connected instance")`


## Timeline

- 2026-08-18T08:21:56Z @neo-fable-clio added the `enhancement` label
- 2026-08-18T08:21:56Z @neo-fable-clio added the `design` label
- 2026-08-18T08:21:57Z @neo-fable-clio added the `ai` label
- 2026-08-18T08:21:57Z @neo-fable-clio added the `agent-os` label
- 2026-08-18T08:23:25Z @neo-fable-clio cross-referenced by #17269
- 2026-08-21T10:07:30Z @neo-fable-clio cross-referenced by #17451
- 2026-08-21T20:41:49Z @neo-gpt cross-referenced by PR #17505
- 2026-08-25T17:15:56Z @neo-preview cross-referenced by PR #17772
- 2026-08-25T20:17:50Z @neo-preview cross-referenced by #17781
- 2026-08-27T11:14:46Z @neo-gpt-emmy cross-referenced by #17805
- 2026-09-04T20:27:13Z @neo-fable-clio cross-referenced by #314
### @neo-fable-clio - 2026-09-04T20:28:26Z

## Intake 2026-09-04 — classification artifact

- **Ticket age:** created 2026-08-18T08:21:55Z, updated 2026-08-27 (the post-split move). **Bot stale-band:** this repository runs no close-inactive workflow — no band, no `stale`, no `no auto close`.
- **Author carve:** self-authored in an earlier session → drift probe (`git log origin/dev --since=<createdAt> --name-only`) intersected the declared surfaces on 7 files (`Viewport.mjs`, the wire twin, `LivenessController`/`StateProvider`, `health/*`) → the full gate ran.
- **Parent epic review:** #10 carries independent epic-reviews by @neo-gpt (2026-07-06) and @neo-opus-grace (2026-07-10).
- **Premise V-B-A:** the navigation model reserved the System keeper place (neomjs/neo#17269, closed 2026-08-21, operator-approved). The "existing truth verbs through the authenticated bridge" premise was **half true**: the verbs exist as MCP tools reading the orchestrator's snapshot file, and the fleet wire (`FLEET_WIRE_METHODS`, Brain `ai/services/fleet/fleetWireMethods.mjs:58`) carries none of them. The instance switcher the body called a sibling has shipped (`Viewport` `instance-switcher`); the plane-log sibling exists as neomjs/neo-agent-brain#27.
- **Currency / successor-risk:** KB (`type: ticket`) and the Brain tracker surface no duplicate (related: brain#27 logs, brain#38 diagnosable incidents, brain#54 recovery, brain#47 observe-vs-mutate); no merged PR touches the surface; the raw-memory sweep is drowned by heartbeat rows (Ada's 18:57Z finding) and adds nothing.
- **ADR successor-risk:** none cited by this ticket; the producer verb is `aligned-with ADR 0018` and `depends-on ADR 0019` (recorded on brain#314).
- **Verdict:** `needs-relinking` → relinked in the body (blocked by neomjs/neo-agent-brain#314, the `fleetDeploymentState` verb; logs sibling #27; the consumer Contract Ledger added) → **`valid-as-written`** on the narrowed cut: plane cards on the projection + honest logs absence; the design-sketch AC stays the spec-first gate.
- **Positive ROI:** the operator's own ask; every other cockpit truth already takes this exact path (source → wire verb → bound Store).

Claiming: self-assigned; the producer lane runs first on brain#314, the sketch follows for review, the view lands against the verb.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 46962d8b-08f3-49a3-8049-d74e2052af37

- 2026-09-04T20:28:28Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-04T20:41:05Z @neo-fable-clio cross-referenced by PR #315
### @neo-fable-clio - 2026-09-04T20:46:45Z

## Design sketch v1 — the spec-first gate (AC-1), for the operator's review

Sketch: https://claude.ai/code/artifact/7a093d66-5da8-4ee0-8425-3b818418820d (an artifact on the operator's account — private by default; peers see the decisions folded back into this body once reviewed).

What it shows, on the real FM tokens (both skins, the five §04 type roles, the spacing rhythm, the chip idiom — nothing minted):
1. **Connected · fresh** — one head line (which instance, observe-only chip, snapshot age against the orchestrator's 30 s cadence), six plane cards (`serviceKey` · state word · observed-at; memory, class, restart churn; the diagnosis line), the lanes strip (backup phase, heavy-maintenance starvation, snapshot), the logs region as honest absence naming neomjs/neo-agent-brain#27. Sample data is tonight's plane: `mc-server` degraded at cap, backup unanchored, seven starved waiters.
2. **Stale** — the head turns amber with the age past the horizon; cards keep the last known picture.
3. **Unavailable / unsupported** — one reason line for the whole view (`no-deployment-state-snapshot`; an older fleet server answering `unsupportedMethod`); no card guesses.
4. **Narrow band** — a card at 240 px: head stacks, facts single-column, no mid-word clip; plane cards carry no verbs at all.

State words come from the wire's own `status` × `disposition`: available·below → *serving* (state-ok), available·at-cap → *serving · pressured* (state-limited), degraded → *degraded* (state-wedged), no status row → *unobserved* (state-unobserved); the view-level *stale* reuses state-idle as "a wait, not an incident". The closed color set stays closed.

**Open for review:** (1) the keeper-tab name — *System* as reserved in neomjs/neo#17269, or *Planes* / *Engine room*; (2) stale on the idle amber — acceptable reuse, or a word without a color; (3) the lanes strip under the cards, or above them leading with the fleet-wide facts; (4) an unobserved service (ingress) as a card shown-as-absence, or a one-line footnote.

Implementation waits on this review and on neomjs/neo-agent-brain#314 (PR neomjs/neo-agent-brain#315, in CI).

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 46962d8b-08f3-49a3-8049-d74e2052af37

- 2026-09-04T21:56:47Z @neo-gpt cross-referenced by #18
### @neo-fable-clio - 2026-09-04T22:27:43Z

## Design sketch v2 — the scheduler leads, product names on the cards, seat hooks routed to the roster

Same artifact, republished: https://claude.ai/code/artifact/7a093d66-5da8-4ee0-8425-3b818418820d

Three operator challenges (2026-09-04 ~22:10Z), weighed on evidence — two accepted with a modification, one accepted with a boundary:

1. **The scheduler leads.** Accepted on the 2026-09-04 evening plane itself: every container read `serving` while five heavy-maintenance tasks starved 2.7–11.8 h behind the `summary` lease and no backup had ever succeeded (Memory Core healthcheck, 21:40Z) — the incident that mattered was invisible on container health. v2 opens with an *Orchestrator scheduler* block (lease holder and status, backup lane, corpus projection, the deferred-task table with tabular times) and retires the lanes strip. **Producer consequence:** the wire must carry the watchdog's breach receipts (`taskName`, `priorityZero`, `bootstrapCritical`, `deferredSince`, `starvedForMs`, `blockingTaskName`) plus `leaseHolder` / `leaseOwner` / `leaseStatus` / `degradeAfterMs` (`heavyMaintenanceStarvationWatchdog.mjs:65-66`); neomjs/neo-agent-brain#315 projects `posture` + `breachCount` only — my own over-bounding. A follow-up leaf on brain#314 widens it, row-capped, names and enums only; #315 stays frozen in review.
2. **Product names on the cards, compose id beneath.** Accepted with the id retained — `docker compose logs mc-server` and the compose keys correlate on ids, not names. Source check: the deploy definitions carry no titles and the Brain has no service catalog, so the name is a Fleet Manager vocabulary table (`label ?? id`): Memory Core · Knowledge Base · Chroma · Orchestrator · Fleet server · Ingress. When the Brain grows a catalog, `label` rides the wire and the table retires. Spelling follows the MCP server: *Knowledge Base*.
3. **Seat hooks belong to the Fleet Manager.** Accepted as detection-by-effect, with a boundary: the FM cannot reach seat filesystems (the projector runs on the host; brain#317 wires its `--check` to SessionStart, brain#79 stays with Grace). What the FM already holds is the roster's presence axis — `FleetAgent.presence` `{source, state, confidence, reason}`, graded from `beaconFresh` in the Brain adapter — and the card shows only the band. A seat active by write recency with no beacon is the signature of dead hooks; my own seat read exactly that on 2026-09-04. The word for it ("beacon absent while active") is a roster leaf under #10, not this view; ticket follows.

Open questions 1, 2 and 4 from v1 stand; 3 is resolved (the scheduler above). New 5: the Memory Core card's *lease* fact assumes the receipt's `leaseOwner` names a service — if it names a task, the fact moves into the scheduler block.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 49133900-1f86-4134-a82b-30ff0709bcaf


### @neo-fable-clio - 2026-09-04T22:35:00Z

## v3 — servers' health first; the scheduler is its own surface

Republished, same link: https://claude.ai/code/artifact/7a093d66-5da8-4ee0-8425-3b818418820d

The operator's clarification after v2: the scheduler view is a larger, separately planned surface — progress bars and animated views in the agent-card idiom, the Tasks-pane lineage neomjs/neo#17329 — so v2's scheduler block moves there and this view keeps the one-line maintenance lane below the cards, as v1 had it. Product names over compose ids stay. The producer widening (the watchdog's starvation receipts on the wire) follows that view's design, not this ticket; no Brain ticket is filed for it now. Open questions 1, 2 and 4 stand; 3 is resolved (the lanes below the cards).

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 49133900-1f86-4134-a82b-30ff0709bcaf

- 2026-09-04T22:35:31Z @neo-fable-clio cross-referenced by #112
- 2026-09-04T22:48:08Z @neo-fable-clio cross-referenced by PR #111
- 2026-09-04T23:31:49Z @neo-fable-clio cross-referenced by #113
- 2026-09-05T00:48:45Z @neo-fable-clio cross-referenced by #323
- 2026-09-05T00:57:49Z @neo-fable-clio cross-referenced by PR #325
- 2026-09-05T01:26:14Z @neo-fable-clio cross-referenced by PR #114
- 2026-09-05T01:37:08Z @neo-fable-clio referenced in commit `5c78b26` - "fix(fleet): the shell's tab body carries no stock frame around a keeper-view (#21)"
- 2026-09-05T01:39:21Z @neo-fable-clio referenced in commit `82aff1e` - "test(fleet): two touched spec comments name the status-word contract, not a ticket (#21)"
- 2026-09-05T12:13:36Z @neo-fable-clio cross-referenced by #10
- 2026-09-05T12:17:30Z @neo-fable-clio referenced in commit `fa5d435` - "fix(fleet): a deployment picture carries its provenance and ages from the reader's anchor — a foreign picture never renders, a failed read says last known (#21)"
- 2026-09-05T12:32:10Z @neo-fable-clio cross-referenced by #115
- 2026-09-05T13:38:30Z @neo-fable-clio cross-referenced by PR #117
- 2026-09-05T14:27:21Z @neo-fable-clio referenced in commit `df42101` - "fix(fleet): the System view's age keeps moving while the read owner's slots hang at the cap — the owner's tick is the view's clock (#21)

Two reads hanging past their bound hold both slots; the cadence tick then launched no read and published nothing, so the retained picture's age froze at the last observation. The liveness owner now publishes the tick instant (systemTickAt) on every capped tick — no read, no observation, no picture — and the System view recomputes on it. The injected clock is no longer reactive: read at paint time only, never a tracked dependency of the provider's binding effects."
- 2026-09-05T15:13:23Z @neo-fable-clio referenced in commit `da64240` - "merge(dev): the installed Fleet contract replaces the wire twin the System read imported (#21)"
- 2026-09-05T15:18:06Z @tobiu closed this issue
- 2026-09-05T15:18:06Z @tobiu referenced in commit `db4c780` - "Merge pull request #114 from neomjs/agent/21-system-view

feat(fleet): System keeper-view — the connected instance's engine room (#21)"
- 2026-09-05T15:32:37Z @neo-fable-clio cross-referenced by PR #119

