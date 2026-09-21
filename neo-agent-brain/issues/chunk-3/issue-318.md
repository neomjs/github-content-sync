---
id: 318
title: 'The presence row says whether a seat''s beacon is fresh, stale or absent'
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-fable-clio
createdAt: '2026-09-04T22:35:18Z'
updatedAt: '2026-09-05T18:24:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/318'
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
closedAt: '2026-09-05T18:24:29Z'
---
# The presence row says whether a seat's beacon is fresh, stale or absent

## Context

The roster's presence axis grades a seat's band from the plane's `who_is_online` verdict plus the turn-presence beacon (`ai/services/fleet/fleetPresenceStateAdapter.mjs:96-117` `beaconFreshAtBound`, `:211`, `:244` `gradePresenceBand`). The beacon is folded INTO the band and then dropped: a consumer cannot tell "online because the seat's hooks beaconed" from "online because `add_memory` wrote recently". On 2026-09-04 the Clio seat had run with no projected hooks for days (the class #317 fixes): `who_is_online` graded it `idle` from write recency with `turnPresence: null`, and every surface stayed green. The Fleet Manager wants to word exactly that case ("beacon absent while active", an Institution leaf) — it needs the facet on the wire.

## The Problem

A hook that fails open is invisible at every surface a human looks at (Grace, 2026-09-04 21:55Z); the one place its failure is observable is the absence of its effect — the beacon — beside evidence that the seat is active. The presence row carries the graded band, `reason` (redacted prose), `validationState`, `since` — no typed beacon fact.

## The Architectural Reality

`fleetPresenceStateAdapter.mjs` builds the per-identity observation from `payload.agents[].signals.turnPresence` (`:211`) and emits `{state, lastSeenAt, reason, validationState, since}` (`:244-250`). `beaconFreshAtBound` already distinguishes absent (`!turnPresence` → false) from stale (horizons elapsed → false) from fresh — the distinction is computed and discarded. On the FM side the presence axis is a passthrough contract (`FleetAgent.presence`, Institution `apps/agentos/model/FleetAgent.mjs:86-94`): a new field reaches the card without re-derivation.

## The Fix

Emit one closed-enum facet on the presence observation: `beacon: 'fresh' | 'stale' | 'absent' | 'unobserved'` — `fresh` / `stale` from the existing horizon evaluation at the snapshot bound, `absent` when the row carries no `turnPresence`, `unobserved` when the presence read itself did not answer (the existing `readReason` path). No change to the band grading; no new reason prose.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| presence observation `beacon` | `beaconFreshAtBound` + the row's `signals.turnPresence` | closed enum, evaluated at the snapshot bound like `beaconFresh` | `unobserved` when the read fails; never inferred from `state` | adapter JSDoc | spec arms per value, incl. absent-vs-stale |
| roster DTO passthrough | the existing presence-axis contract (`{source, state, confidence, …}`) | the field rides the row untouched | field absent = older Brain, the consumer keeps its fallback | DTO docblock | DTO arm |

## Decision Record impact

aligned-with the presence contract graduated from D#16720 (#53: three independent signals; a tier that cannot answer renders absence, never a verdict) — the facet is a fourth fact on the same axis, not a fused verdict.

## Acceptance Criteria

- [ ] `fleetPresenceStateAdapter` emits `beacon` per identity with the four values above; `fresh` / `stale` agree with `beaconFreshAtBound` at the same bound; a row without `turnPresence` is `absent`; a failed presence read is `unobserved` for every row.
- [ ] The band grading is byte-identical before and after (existing spec arms unchanged).
- [ ] The cockpit roster DTO carries the field through unchanged; `fleetPresenceStateAdapter.spec` covers each value and the absent-vs-stale distinction red-first.
- [ ] No reason prose is derived from the facet Brain-side; wording stays with the consumer.

## Out of Scope

The Fleet Manager card word (Institution leaf, follows this); hook projection and self-check (#317, #79); the second-session visibility gap (#287).

## Related

#53 (presence contract), #317, #79, #287; Institution #10 (consumer epic), Institution #21. Live latest-open sweep: latest 20 open Brain issues at 2026-09-04 ~22:25Z, no equivalent; A2A last-30 all-states at 22:28Z, no claim on this surface; Memory Core recall: the unscoped query drowned in boot-noise hits — null, not clean; own-assignment sweep: #53's open remainder is viewer scoping, not this facet.

Origin Session ID: 49133900-1f86-4134-a82b-30ff0709bcaf
Retrieval Hint: "presence row beacon facet fresh stale absent unobserved fleetPresenceStateAdapter"

## Timeline

- 2026-09-04T22:35:18Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-04T22:35:19Z @neo-fable-clio added the `enhancement` label
- 2026-09-04T22:35:19Z @neo-fable-clio added the `ai` label
- 2026-09-04T22:35:19Z @neo-fable-clio added the `agent-os` label
- 2026-09-05T00:48:45Z @neo-fable-clio cross-referenced by #323
- 2026-09-05T00:54:44Z @neo-fable-clio cross-referenced by #324
- 2026-09-05T15:09:43Z @neo-fable-clio cross-referenced by PR #332
- 2026-09-05T15:32:37Z @neo-fable-clio cross-referenced by PR #119
- 2026-09-05T18:24:29Z @tobiu referenced in commit `3f8bf86` - "Merge pull request #332 from neomjs/agent/318-presence-beacon-facet

feat(fleet): the presence row carries its beacon facet — fresh · stale · absent · unobserved (#318)"
- 2026-09-05T18:24:29Z @tobiu closed this issue

