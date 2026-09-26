---
id: 239
title: 'The app seeds nothing: the sample roster and activity retire, cold and empty states are the surfaces'' own'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-09-26T09:24:43Z'
updatedAt: '2026-09-26T13:43:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/239'
author: neo-fable-clio
commentsCount: 0
parentIssue: 237
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-26T13:43:37Z'
---
# The app seeds nothing: the sample roster and activity retire, cold and empty states are the surfaces' own

## Context

Second leaf of #237, after the tests own the fixtures. The operator's ruling of 2026-09-26: "either there is real data, or there is not." The team shell attached to the plane showed eleven invented agents and six invented events under "fleet offline · showing the static roster" while the registry was empty and the transport was up.

## The Problem

`sample` is a data claim dressed as a state: the roster's head reads "static roster", the stream's "sample · live feed pending", and the banner composes "showing the static roster" onto every cold verdict. None of it tells the operator what the read answered. A cold surface (no answer yet), an empty answer (the registry has no agents; nothing happened yet) and a failed transport are three different truths, and the cockpit has words for none of them because the sample stood in for all three.

## The Architectural Reality

- Seeds: `apps/agentos/config/fleetSampleData.mjs`, `apps/agentos/resources/data/fleetRoster.json`; the roster store's `autoLoad` on that url; the activity store seeded from the module by the provider.
- Vocabulary: `StateProvider` defaults `gridAdapterState: 'sample'` / `streamAdapterState: 'sample'`; `TargetBinding` republishes `sample` on rebinding; `roster/Container#applyAdapterState` ('static roster'); `activity/Container#updateHeader` ('sample · live feed pending'); `SpineBanner` verdicts on `state === 'sample'` (lines ~230–300, "Fleet data unavailable — showing the static roster · <reason>"); the cockpit's `rosterSourceMode: 'sample' | 'selected'`.
- The roster's bootstrap CTA "Add your first agent" (`roster/Container.mjs` ~171–193, a design ruling on record: an empty fleet has a findable path to its first agent) — the empty state already exists and the sample was painted over it.
- The harness witness `harness/adapterWitness.mjs`: `ADAPTER_STATES` includes `sample`, `ROSTER_STATE_LABELS` / stream labels map it, the product witness requires `cardCount > 0`; `harness/main.mjs` reads the first-paint report through it.
- The visual goldens booted on the sample (`cockpit-default-shell`, `cockpit-intermediate-720`, `cockpit-review-1280`, the roster and activity captures) — after this leaf the cold captures show the empty states, and captures that need rows land the fixture (the first leaf's driver).

## The Fix

1. Delete the two seed files; the roster store starts empty (no `url`), the activity store starts empty; `rosterSourceMode` loses `sample`.
2. Adapter vocabulary: `cold` (no answer yet) · `live` · `stale` · `degraded`; `sample` gone everywhere (provider defaults, `TargetBinding`, both surfaces, `SpineBanner`, the witness). A live answer with zero rows renders the surface's empty state with the live word.
3. Empty and cold states in the surfaces' own words: the roster shows the CTA when its answer is empty and a quiet "not answered yet" line while cold; the activity stream shows "no activity yet" / "not answered yet" in the same roles; head classes `is-cold` / `is-empty` for the stylesheets.
4. The banner keeps to transport verdicts: "connecting", "fleet server unreachable — <reason>", "reconnecting"; "showing the static roster" and "fleet registry empty" leave the banner (the CTA says the latter).
5. The witness: label maps without `sample`; the product witness passes on real cards OR the empty CTA present (`firstPaint.emptyCta`), never on a count of invented cards; the smoke's first-paint report gains that field.
6. Goldens: the cold cockpit captures re-captured (empty states); captures with rows land the fixture first (leaf one's driver); the stamp re-issued; specs on the vocabulary updated.

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `gridAdapterState` / `streamAdapterState` (provider leaves) | this ticket (`cold` · `live` · `stale` · `degraded`) | no `sample` value exists | — | `StateProvider` JSDoc | AC-2 |
| `SpineBanner` verdict text | this ticket | transport verdicts only | — | `SpineBanner.mjs` docblock | AC-3 |
| `HARNESS_FIRST_PAINT_REPORT` (`adapterWitness.mjs`) | this ticket | `emptyCta` joins the report; the product witness passes on cards or the CTA | — | `adapterWitness.mjs` docblock | AC-4 |

## Decision Record impact

none — the sample's own JSDoc was the prior ruling; the operator's 2026-09-26 words supersede it (recorded on #237).

## Acceptance Criteria

- [ ] AC-1 `apps/agentos/config/fleetSampleData.mjs` and `apps/agentos/resources/data/fleetRoster.json` are gone; the stores start empty; no fetch of a roster JSON at boot (NL arm: the roster store's `count` is 0 and its `url` null before any landing).
- [ ] AC-2 `grep -rn "'sample'" apps/agentos harness` returns only the tasks pane's own rows (the third leaf, #240); the roster, the stream, the banner, the provider and the witness speak cold · live · stale · degraded (unit specs on the provider, `TargetBinding`, both surfaces). *(Restated 2026-09-26: the first wording contradicted Out of Scope, which keeps the tasks pane's sample rows for #240.)*
- [ ] AC-3 Cold: the roster shows "not answered yet" and no cards, the stream "not answered yet"; a live empty answer: the roster shows the CTA "Add your first agent", the stream "no activity yet"; the banner never mentions the static roster or an empty registry (unit specs per state; the NL arm lands a live empty answer through the driver).
- [ ] AC-4 The packaged smoke on the rebuilt `.app` passes its product witness with `cardCount: 0` and `emptyCta: true`; `adapterWitness.spec.mjs` covers the new conjunct.
- [ ] AC-5 Visual: the cold captures show the empty states in both skins; captures with rows use the driver; stamp re-issued; the suite green.
- [ ] AC-6 (post-merge) The team shell in plane-attach shows the roster's CTA and "no activity yet" instead of the sample (Neural Link read).

## Out of Scope

The tasks pane's sample rows (the third leaf); the plane-side roster composition (Brain); the Add-agent flow.

## Avoided Traps

- A "demo mode" flag (a seed with a switch); re-labelling the seed; keeping it for goldens (leaf one exists so no golden needs the app's seed).

## Related

#237 (parent), the first leaf (the fixtures), #13 (the tokens the empty states wear), #11, neomjs/neo-agent-brain#83 / #53 (where the real roster comes from).

Live latest-open sweep: the latest 20 open issues at 2026-09-26T09:22:32Z — no equivalent. A2A sweep: none (Ada informed). Memory Core sweep: the seed's JSDoc. Own-assignment sweep: #237 only. Structure map: N/A.

Origin Session ID: 26b775fe-f8d9-4258-809c-09d9e5ef8ed1
Retrieval Hint: `query_raw_memories("cockpit adapter vocabulary cold live stale degraded sample retired empty CTA")`


## Timeline

- 2026-09-26T09:24:43Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-26T09:24:44Z @neo-fable-clio added the `enhancement` label
- 2026-09-26T09:24:44Z @neo-fable-clio added the `agent-os` label
- 2026-09-26T09:24:45Z @neo-fable-clio added the `ai` label
- 2026-09-26T09:24:45Z @neo-fable-clio added the `design` label
- 2026-09-26T09:25:16Z @neo-fable-clio added parent issue #237
- 2026-09-26T09:34:23Z @neo-opus-ada cross-referenced by #241
- 2026-09-26T09:34:26Z @neo-opus-ada cross-referenced by #242
- 2026-09-26T09:34:32Z @neo-opus-ada cross-referenced by #246
- 2026-09-26T10:04:50Z @neo-fable-clio cross-referenced by #249
- 2026-09-26T10:08:15Z @neo-fable-clio cross-referenced by PR #250
- 2026-09-26T12:44:06Z @neo-fable-clio cross-referenced by PR #254
- 2026-09-26T12:46:32Z @neo-fable-clio referenced in commit `ddded48` - "test(agentos): the burst spec's row-geometry comment names its witness, not a ticket (#239)"
- 2026-09-26T13:07:29Z @neo-fable-clio referenced in commit `aed598a` - "merge(238): re-stack onto #250's repaired head — the admission is FleetAdmission's, the empty answer stays authoritative, the full-page goldens carry the Observatory rail entry (#239)"
- 2026-09-26T13:07:30Z @neo-fable-clio referenced in commit `c6124b8` - "merge(238): the baseline stamp follows #250's stamp commit (#239)"
- 2026-09-26T13:13:25Z @neo-fable-clio referenced in commit `8624d3f` - "test(agentos): the roster refill seam drives a switch's clear · clear + add — nothing is seeded, so nothing reloads (#239)"
- 2026-09-26T13:43:37Z @tobiu referenced in commit `b55d9a5` - "Merge pull request #254 from neomjs/clio/239-no-seed

feat(cockpit): the app seeds nothing — the sample fleet retires, the surfaces speak cold · live · stale and render their own empty states (#239)"
- 2026-09-26T13:43:38Z @tobiu closed this issue

