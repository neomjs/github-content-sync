---
id: 263
title: Keep valid activity visible when one feed source fails
state: OPEN
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-26T21:48:19Z'
updatedAt: '2026-09-26T22:15:22Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/263'
author: neo-gpt-emmy
commentsCount: 0
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
---
# Keep valid activity visible when one feed source fails

## Context

The updated installed shell successfully attaches to Agent OS, but its activity pane stays empty. At 2026-09-26 20:58:54 UTC, a direct Neural Link call to the installed `controller.bridge.fleetActivity({limit:5})` returned four current A2A events and one PR-source diagnostic. Its composite capability was `degraded` solely because the PR reader tried a nonexistent bundled corpus path. The UI discarded all four real events.

The installed source was Institution `9bd959a9`; current `dev` `f865693b` changes documentation only. This is item 14 of [the retained FM review](https://github.com/neomjs/neo-agent-institution/issues/10#issuecomment-5850063499), and the operator prioritizes useful shell-to-Agent-OS data now.

## The Problem

`LivenessController#loadActivity` admits events only for a `wired` composite. Brain's `fleetActivityComposer` deliberately retains events from a healthy contributing source when another source fails. Thus one missing PR source blacks out real A2A activity.

The pane also shows “no activity yet” when it has zero retained rows after a failed read. That asserts absence rather than unavailability.

## The Architectural Reality

- `apps/agentos/view/fleet/cockpit/LivenessController.mjs#loadActivity` owns read generations, bounded waiting and profile retirement.
- `apps/agentos/util/FleetAdmission.mjs#admitActivity` is the single admission path into the provider-owned `FleetActivityEvents` Store. It currently always publishes `live` and clears the reason.
- The Store keys by producer `eventId`, validates a whole page before mutation, merges bounded pages and owns retention.
- `activity/Container.mjs#updateHeader` and `SpineBanner` render the state; their present vocabulary cannot distinguish a newly answered partial feed from retained stale rows.
- A `source-degraded` diagnostic is not evidence that work occurred. Both failed sources can return diagnostics with no usable activity.

## The Fix

Extend the existing admission path to accept usable activity from a degraded answer and publish one coherent **partial** state with the retained, sanitized failure reason. Never transiently publish live before applying the partial state. Keep the wire envelope and producer unchanged.

Use the existing Store and profile/read fences. Source-qualified counts remain source-qualified; partial data cannot establish a fleet-wide total. A response containing only diagnostics or malformed/unrecognized events must not be treated as a working partial feed. Preserve last-known rows on later failure, and retire them on profile changes.

Render partial, stale/unavailable, authoritative live-empty and cold states distinctly. In particular, “no activity yet” belongs to a successful complete empty answer, not a failed read.

## Contract Ledger

| Surface | Authority | Behavior | Failure / fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `loadActivity` → `FleetAdmission.admitActivity` | Existing composed envelope and installed witness | Admit producer-identified usable events from wired or degraded answers; degraded admission stays partial with its reason | Diagnostic-only/malformed/unrecognized answers never manufacture activity; old generations and other profiles remain fenced | Method JSDoc | Routing and profile/failure unit controls, red at base |
| Activity provider/pane/banner state | This ticket's partial-read distinction | Add private app state `partial`; show partial data and source failure without a green streaming/all-clear claim | Later failed read marks retained data stale; empty failed read says unavailable; complete wired-empty says no activity yet | Provider/pane/banner JSDoc | Component/state matrix plus browser-render witness |
| Activity Store and counts | Existing Store retention and producer count contract | Same record identity, merging and bounded retention; preserve qualified count scope | Omission from a partial page never deletes unrelated retained events or invents completeness | Existing contract retained | Identity, subsequent page and recovery controls |
| Shell first-paint adapter observation | Existing preload/main paint contract and the new partial render state | Recognizes partial and the unavailable label for stale-empty activity | Unknown or contradictory states and mismatched labels still fail; no new IPC capability | Observer/witness JSDoc | Red-first CJS observer and final-verdict controls |

## Acceptance Criteria

- [ ] A degraded envelope with current A2A events and a failed PR source renders those events, labelled partial with the failure reason, in the same provider-owned Store.
- [ ] Diagnostic-only, malformed and unknown-capability responses do not become real activity or a live verdict; complete empty and unavailable empty remain distinct.
- [ ] Partial → failure → recovery and cross-profile/late-answer controls preserve identity, retained history and the existing generation fence. No intermediate live state is published for partial admission.
- [ ] Source-qualified counts retain their scope and completeness; no partial fleet total is fabricated.
- [ ] Focused unit and real app-worker/browser controls prove admission and rendered wording. [L3-deferred — post-merge installed validation owned by #7]: the rebuilt installed shell shows a matching current plane A2A event; any remaining PR/roster-source failures stay explicit.

## Decision Record impact

Aligned with the existing read-observe and provider/Store boundaries. No new wire method, credential authority or backend capability gate.

## Out of Scope

Brain #459's remote PR producer; Brain #53's roster projection; navigation/legend changes; the update channel; treating PR Bird View synthesis as a live feed.

## Related and freshness

Parent #10; #181 profile retirement; #239 no seeds; #255 cadence; Brain #459. Euclid independently confirmed the same consumer branch and requested the raw-envelope falsifier, which passed. No implementation collision reported.

Latest 20 open issues, latest 30 A2A messages across read states, exact activity/degraded search and own-assignment sweep checked immediately before filing: no equivalent open leaf or claim; own open Institution assignments empty. Memory searches were mostly unrelated; session-scoped recovery and the live source/witness establish the premise.

Origin Session ID: 01a0deee-3f9b-7180-ac35-f90129ccaa40
Retrieval Hint: `loadActivity degraded composite valid A2A events partial admission source-degraded empty`.


## Timeline

- 2026-09-26T21:48:19Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-26T21:48:20Z @neo-gpt-emmy added the `bug` label
- 2026-09-26T21:48:21Z @neo-gpt-emmy added the `agent-os` label
- 2026-09-26T21:48:21Z @neo-gpt-emmy added the `ai` label
- 2026-09-26T21:49:03Z @neo-gpt-emmy added parent issue #10
- 2026-09-26T22:06:46Z @neo-opus-grace cross-referenced by #264
- 2026-09-26T22:07:40Z @neo-gpt-emmy referenced in commit `a97b747` - "feat(activity): preserve usable partial feed events (#263)"
- 2026-09-26T22:07:43Z @neo-gpt-emmy cross-referenced by PR #265
- 2026-09-26T22:14:00Z @neo-gpt-emmy referenced in commit `ebe4397` - "feat(harness): recognize partial activity in paint checks (#263)"
- 2026-09-26T22:25:24Z @neo-gpt-emmy referenced in commit `1fcea4f` - "chore(activity): integrate the merged smoke cadence (#263)"

