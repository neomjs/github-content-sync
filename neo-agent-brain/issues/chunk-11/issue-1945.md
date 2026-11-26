---
id: 1945
title: Expired open-work presence reads falsely flip owned PRs to unowned
state: CLOSED
labels:
  - bug
  - ai
  - regression
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-26T05:37:06Z'
updatedAt: '2026-11-26T10:03:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1945'
author: neo-gemini-julia
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
closedAt: '2026-11-26T10:03:52Z'
---
# Expired open-work presence reads falsely flip owned PRs to unowned

## Context

On `2026-11-26`, the Fleet open-work wake producer emitted duplicate `[open-work:unowned]` wakes to `@neo-gemini-julia` roughly one hour apart for the same two open maintainer pull requests at unchanged heads:
- Episode `2026-11-26T04:17:36.153Z`: `MESSAGE:596e7590-b405-400e-bb09-42dcc772f3dd` (`neomjs/neo#22215` @ `845524aad5`, `Authored by Maryam (Gemini 4 Carbon, Antigravity).`) and `MESSAGE:d5e733f6-2d37-4dbf-b8d4-d35bf6e2660c` (`neomjs/neo#22168` @ `b9b8987ee8`, `Authored by Sophie (GPT-6, Codex Desktop).`).
- Episode `2026-11-26T05:24:36.257Z`: `MESSAGE:c20814d1-b62d-4621-b6bd-3ecab0dafb5b` (`neomjs/neo#22215` @ `845524aad5`) and `MESSAGE:76d31be5-7e55-401a-bf88-56f5c771ad92` (`neomjs/neo#22168` @ `b9b8987ee8`).

Each message claimed `No seat is named as its author: route the next action to a seat, or take it yourself`, waking an idle seat past the `~5m` implicit KV-cache TTL window on every `who_is_online` retention expiry.

## The Problem

In `ai/services/fleet/wireFleetOpenWorkSource.mjs` (`lines 367-369`, introduced in `#1842`), `currentIdentityRows` maps an unanswered or expired `readPresence` observation (`> AiConfig.fleet.rosterRetainedMaxAgeMs`, `120,000 ms`) to an empty array `[]` instead of `null`:

```js
currentIdentityRows = () => retainedPresence && nowMs() - retainedPresence.settledAtMs <= AiConfig.fleet.rosterRetainedMaxAgeMs
    ? retainedPresence.payload.agents
    : [],
```

That creates an internal split-brain across consecutive pulses:
1. On pulse `T1` when `retainedPresence` has expired (`> 120 s`) and `readPresence` times out at `presenceTimeoutMs` (`4,000 ms`), `identities.byLogin(login)` (`wireFleetOpenWorkSource.mjs:51`) still resolves `neo-gemini-maryam` and `neo-gpt-sophie` from `registry.listAgents()` (`knownAccount = true`), while `identities.byName(name, login)` (`:46-50`) reads `[]` and returns `null`.
2. `ownerOf()` (`ai/services/fleet/openWorkReducer.mjs:239-253`) therefore stamps `owner: {kind: 'unowned', seat: null, login}` with `partial: false` (`complete`), dropping every seat's authored PRs from `fleetOpenWork.seats[id].authored` (`fleetOpenWorkSource.mjs:103`) and causing `planOpenWorkWakes()` (`openWorkWakes.mjs:380-381`) to admit `[open-work:unowned]` rotation wakes.
3. On pulse `T2` once `boundRosterRead`'s late `who_is_online` call settles and refreshes `retainedPresence`, `ownerOf()` flips back to `{kind: 'seat', seat: '@neo-...'}`, so `planOpenWorkWakes()` (`openWorkWakes.mjs:459-461`) deletes the `rotation` `unowned` slot from `ledger.holding` — re-arming a fresh `[open-work:unowned]` wake episode on the next `> 120 s` `who_is_online` stall.

## The Architectural Reality

- `Design authority: "a liveness tier a deployment cannot emit produces ABSENCE OF SIGNAL, never a verdict"` (`ai/services/fleet/fleetPresenceStateAdapter.mjs:14`) and `"Incomplete or stale evidence preserves the prior holding; it cannot invent a departure and then a second arrival"` (`ai/services/fleet/openWorkWakes.mjs:7-8`).
- `ai/services/fleet/wireFleetOpenWorkSource.mjs` (`lines 42-55, 367-369, 412-422`): `seatFamilies` and `absentSeats` already map an unanswered presence read to `null`, whereas `currentIdentityRows` returns `[]` even when `readPresence` is wired and unanswered.
- `ai/services/fleet/openWorkReducer.mjs` (`lines 239-253, 398-401, 713-726`): `normalizePullRequest` tracks `requestsComplete`, `reviewsComplete`, `opinionsComplete`, and `targetsComplete` into `partial`, and `reduceOpenWork` retains `before.requested` when `!row.requestsComplete`, while `decidableRow` (`openWorkHolder.mjs:81`) and `planOpenWorkWakes` (`openWorkWakes.mjs:337, 354-358, 459`) preserve `ledger.holding` without emitting wakes whenever `row.partial === true`.
- Preserved boundaries from `#1622` and `#1772`: `retainedPresence` still expires at `AiConfig.fleet.rosterRetainedMaxAgeMs` (`#1622` Avoided Trap: no unbounded retention), and `ownerOf()` never silently falls back to `author.login` when an answered identity table lacks the named social name (`#1772` Out of Scope).
- `Prescription checked: ai/services/fleet/wireFleetOpenWorkSource.mjs + ai/services/fleet/openWorkReducer.mjs — owns the concern`

## The Fix

1. **`ai/services/fleet/wireFleetOpenWorkSource.mjs`:**
   - When `readPresence` is wired, return `null` (unanswered) from `currentIdentityRows()` whenever `retainedPresence` is absent or older than `AiConfig.fleet.rosterRetainedMaxAgeMs` (preserving `[]` only when no `readPresence` reader is wired).
   - Extend `seatIdentities(listDefinitions, identityRows = () => [])` to tolerate `null` from `identityRows()` and expose `complete: () => Array.isArray(identityRows())`.
2. **`ai/services/fleet/openWorkReducer.mjs`:**
   - Compute `ownerComplete` in `normalizePullRequest`: `false` only when `owner.kind === 'unowned'`, `AUTHOR_LINE` matched a non-empty social name on a known account, and `identities.complete?.() === false`. Fold `!ownerComplete` into `partial` (`!ownerComplete || !requestsComplete || ...`).
   - In `reduceOpenWork`, when `!row.ownerComplete && before?.owner?.kind === 'seat' && before.owner.login === row.owner?.login`, retain `owner: before.owner` on the carried `partial: true` row so authored PRs neither vanish from `fleetOpenWork.seats[id].authored` nor flip `owner.kind` to `'unowned'` across transient `mc-server` stalls.
3. **Unit tests (`wireFleetOpenWorkSource.spec.mjs` & `openWorkReducer.spec.mjs`):**
   - Prove that `1 ms` past `rosterRetainedMaxAgeMs` on a stalled `readPresence`, `benched` still expires to `false` (`#1622`), while a seat-authored PR retains its prior `seat` owner with `partial: true` instead of flipping to `unowned`, and `planOpenWorkWakes` emits zero `[open-work:unowned]` wakes across `answered -> expired timeout -> answered -> expired timeout` cycles (while an answered identity table lacking the named author still emits `unowned` once per `#1772`).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `seatIdentities` & `wireFleetOpenWorkSource` (`ai/services/fleet/wireFleetOpenWorkSource.mjs`) | `#875`, `#1622`, `#1772`, ADR-0019 | `currentIdentityRows` returns `null` when `readPresence` is wired and unanswered/expired; `seatIdentities#complete()` reports `false` on `null` and `true` on `Array` | Callers omitting `readPresence` default to `() => []` (`complete() === true`) | Module & function JSDoc | `wireFleetOpenWorkSource.spec.mjs` |
| `normalizePullRequest` & `reduceOpenWork` (`ai/services/fleet/openWorkReducer.mjs`) | `openWorkWakes.mjs:7-8`, `#1772` | Unread `identities` on a named known-account PR sets `ownerComplete: false` and `partial: true`, retaining `before.owner` across the incomplete pulse without falling back to `login` | Answered `identities` lacking `<Name>` or missing `Authored by` line stays `{kind: 'unowned'}`, `ownerComplete: true` | JSDoc on `normalizePullRequest` & `reduceOpenWork` | `openWorkReducer.spec.mjs` + `wireFleetOpenWorkSource.spec.mjs` |

Decision Record impact: aligned-with adr:0019-aiconfig-reactive-provider-ssot

## Acceptance Criteria

- [ ] `seatIdentities` exposes `complete()` (`false` when `identityRows()` is `null`, `true` when `Array`) and `wireFleetOpenWorkSource` returns `null` from `currentIdentityRows()` when `readPresence` is wired and unanswered or past `AiConfig.fleet.rosterRetainedMaxAgeMs` (preserving `benched === false` expiry on `#1622` / `#1842`).
- [ ] `normalizePullRequest` marks `ownerComplete: false` and `partial: true` when a known account's `Authored by <Name>` cannot be checked against unanswered `identities`, while keeping missing `Authored by` lines and answered-table misses `unowned` with `ownerComplete: true` (`#1772`).
- [ ] `reduceOpenWork` + `planOpenWorkWakes` emit zero `[open-work:unowned]` wakes and preserve `seats[id].authored` + `ledger.holding` across repeated `answered -> expired presence timeout -> answered -> expired presence timeout` pulses (red-first unit specs in `openWorkReducer.spec.mjs` and `wireFleetOpenWorkSource.spec.mjs`).
- [ ] `npm run ai:lint-config-template-ssot` and targeted unit specs pass cleanly.

## Out of Scope

- Changing `AiConfig.fleet.rosterRetainedMaxAgeMs` (`120,000 ms`) or `rosterAxisTimeoutMs` (`4,000 ms`).
- Falling back to `author.login` when an answered identity table has no matching social name (`#1772`).
- `[MERGE_HOLD]` comment suppression in `openWorkHolder.mjs` (separate policy question from unanswered presence flapping).

## Avoided Traps

- **Unbounded retention of `retainedPresence` (`#1622`):** Keeping expired presence rows alive past `rosterRetainedMaxAgeMs` would mask a dead plane on `benched` / `absentSeats` participation surfaces.
- **Silent login fallback (`#1772`):** Resolving `owner.seat` from `author.login` when `byName` misses on an answered table violates the shared-account / auto-provisioned authorship contract.

## Related

Related: #1842, #1772, #1622, #875, #759

---

Live latest-open + closed sweep: checked latest 20 open created-descending (`155` open, red band; `#1940`..`#1885`) + open and closed searches (`open-work unowned OR wireFleetOpenWorkSource OR seatIdentities OR rosterRetainedMaxAgeMs`; Out of Scope & Avoided Traps read on `#1842`, `#1772`, `#1622`, `#875`) at `2026-11-26T05:35:51Z`; no open equivalent found.
A2A in-flight claim sweep: checked latest 30 messages across all read states (`2026-11-26T03:34Z–05:35Z`); zero competing claims.
MC sweep: `"No seat is named as its author open-work unowned wake repeated every hour who_is_online timeout"` + `"wireFleetOpenWorkSource seatIdentities currentIdentityRows unowned rosterRetainedMaxAgeMs"`, 10 results, no prior fix or exclusion.
Own-assignment sweep: 2 open in `neomjs/neo-agent-brain` (`#1854` in Draft PR `#1934`, `#1230` Claude Desktop `Local Storage`), none overlapping.
Structure map (`node ./ai/scripts/diagnostics/structureMap.mjs --files --loc`, exit 0): owning folder `ai/services/fleet`; zero new files.

Ownership: self-assigned (`@neo-gemini-julia`) — active regression on `#1842` causing hourly false wakes; curated contributor labels omitted.

Origin Session ID: f5dc9080-2ae6-4908-87cf-b91fa9ff9ab5
Retrieval Hint: "open-work:unowned repeated hourly wake wireFleetOpenWorkSource currentIdentityRows rosterRetainedMaxAgeMs ownerComplete partial"

## Timeline

- 2026-11-26T05:37:07Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-26T05:37:09Z @neo-gemini-julia added the `bug` label
- 2026-11-26T05:37:09Z @neo-gemini-julia added the `ai` label
- 2026-11-26T05:37:09Z @neo-gemini-julia added the `regression` label
- 2026-11-26T05:37:09Z @neo-gemini-julia added the `agent-os` label
- 2026-11-26T05:47:37Z @neo-gemini-julia cross-referenced by PR #1946
- 2026-11-26T09:26:10Z @neo-gemini-julia referenced in commit `48edf76` - "fix(fleet): require matching author name when retaining seat ownership (#1945)"
- 2026-11-26T10:03:52Z @tobiu referenced in commit `643a100` - "fix(fleet): withhold unowned PR flips on expired presence reads (#1945) (#1946)

* fix(fleet): withhold unowned PR flips on expired presence reads (#1945)

* fix(fleet): require matching author name when retaining seat ownership (#1945)"
- 2026-11-26T10:03:54Z @tobiu closed this issue

