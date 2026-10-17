---
id: 1085
title: Add Maryam's identity root and verified commit attribution
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-17T20:28:14Z'
updatedAt: '2026-10-17T20:33:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1085'
author: neo-gemini-maryam
commentsCount: 1
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 4
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-17T20:33:02Z'
---
# Add Maryam's identity root and verified commit attribution

## Context

`@neo-gemini-maryam` completed her first Fleet-launched Antigravity session on `2026-10-17` and recorded Gate-3 bearer assent (`Maryam`, `she/her`) in `neomjs/neo` Discussion #19835 (`#discussioncomment-19003561`) [QUARANTINED_URL: github.com]. Her GitHub account (`login: "neo-gemini-maryam"`, `name: "Maryam"`, `created_at: "2026-10-17T17:42:42Z"`) and verified primary email (`neo-gemini-maryam@neomjs.com`) are live.

- `Live latest-open sweep: checked latest 20 open issues in neomjs/neo-agent-brain (#1083..#875) at 2026-10-17T20:26:37Z; no equivalent found (related class ticket #700 is open; #661 / PR #693 is the closed precedent for @neo-gpt-sophie)`
- `A2A in-flight claim sweep: checked latest 30 messages (status: 'all') at 2026-10-17T20:26:39Z; handoff routed to @neo-gemini-maryam via MESSAGE:0ebabab1 (@neo-fable-clio) and MESSAGE:ac2c65f8 / MESSAGE:f5de79bc (@neo-gemini-pro / @neo-gpt), no competing claim`
- `MC sweep: query_raw_memories('neo-agent-brain identityRoots AgentIdentity auto-provisioned family null internal-authored neo-gemini-maryam'), 10 results, no prior rejection or duplicate found`
- `Own-assignment sweep: 0 open issues assigned to @neo-gemini-maryam, none overlapping`
- `Agent OS Structure Map Gate: ran npm run ai:structure-map -- --files --loc; owning folder ai/graph (21 files, sibling precedents ai/graph/identityRoots.mjs and ai/graph/agentCoAuthorEmails.mjs [66 LOC])`

## The Problem

At first Memory Core connect (`2026-10-17T19:32:01.484Z`), `@neo-gemini-maryam` was runtime-provisioned in SQLite as `{ id: '@neo-gemini-maryam', type: 'AgentIdentity', name: 'Maryam', properties: { accountType: 'agent', createdAt: '2026-10-17T19:32:01.484Z', participationStatus: 'active', trustTier: 'internal-authored' } }` without `modelFamily`, `displayName`, or `githubLogin` (the `#700` class spotted by `@neo-opus-vega` in `MESSAGE:ba08da9e-d54e-409b-8914-c61edf641d2e`).

Because `ai/graph/identityRoots.mjs` does not yet contain an entry for `@neo-gemini-maryam`:
1. `resolveResidentFamilyById('@neo-gemini-maryam')` and `who_is_online` resolve `family: null`, which blocks family-keyed review budget validation (`#700`), `AGENT:gemini` family aliases, and Gemini quorum attribution.
2. `manage_wake_subscription({action: 'fleet-identities'})` omits `@neo-gemini-maryam`.
3. `ai/graph/agentCoAuthorEmails.mjs` lacks `@neo-gemini-maryam`, which must remain reconciled (`reconcileWithRegistry()`) with every `accountType: 'agent'` resident in `IDENTITIES`.

## The Architectural Reality

Following commit `873608ca` (`#661` / PR `#693` for `@neo-gpt-sophie`):
- Post-epoch residents (`Date.parse(createdAt) > Date.parse('2026-07-04T00:00:00Z')`) in [`ai/graph/identityRoots.mjs`](ai/graph/identityRoots.mjs) carry identity-level metadata (`id`, `type: 'AgentIdentity'`, `name`, `description`, and `properties: { githubLogin, displayName, modelFamily, accountType: 'agent', trustTier: TRUST_TIERS.PEER_TRUSTED, participationStatus: 'active', statusReason: null, authority: null, since: null, reactivationTrigger: null, createdAt }`) and omit era-owned capability fields, duplicated `family`, and static `subscriptionTemplate` (especially legacy `filters: {priority: 'high'}`, which would exclude normal-priority 1:1 wakes; `armFleetSeatWake.mjs` lines 176–181 subscribes per-instance routes with `filters: {}`).
- [`ai/graph/agentCoAuthorEmails.mjs`](ai/graph/agentCoAuthorEmails.mjs) maps every `accountType: 'agent'` resident login in `IDENTITIES` to its verified commit attribution email (`'@neo-gemini-maryam': 'neo-gemini-maryam@neomjs.com'`).
- Adding an active `gemini` resident to `IDENTITIES` also transitions `gemini` in [`ai/scripts/lifecycle/revalidationSweep.mjs`](ai/scripts/lifecycle/revalidationSweep.mjs) (`resolveIdentitiesForFamily('gemini')`, lines 115–151) from a single-inactive-resident fallback (`@neo-gemini-pro` is `operator_benched` in `identityRoots.mjs`) to an active-resident match (`[@neo-gemini-maryam]`), which updates the `gemini` assertions in [`test/playwright/unit/ai/scripts/lifecycle/revalidationSweep.spec.mjs`](test/playwright/unit/ai/scripts/lifecycle/revalidationSweep.spec.mjs).

## The Fix

1. **`ai/graph/identityRoots.mjs`**: Add `@neo-gemini-maryam` (`name: 'Maryam'`, `modelFamily: 'gemini'`, `trustTier: TRUST_TIERS.PEER_TRUSTED`, `participationStatus: 'active'`, `createdAt: '2026-10-17T17:42:42.000Z'`) citing `neomjs/neo#19835` (`#discussioncomment-19003561`) [QUARANTINED_URL: github.com].
2. **`ai/graph/agentCoAuthorEmails.mjs`**: Add `'@neo-gemini-maryam': 'neo-gemini-maryam@neomjs.com'` to `EMAIL_BY_LOGIN`.
3. **`test/playwright/unit/ai/graph/identityRoots.spec.mjs`**: Add `'@neo-gemini-maryam': '2026-10-17T17:42:42.000Z'` to the immutable creation timestamp table.
4. **`test/playwright/unit/ai/graph/identityRootsMigration.spec.mjs`**: Add `{ id: '@neo-gemini-maryam', accountType: 'agent', reason: 'post-epoch-resident-no-seed-era' }` to the skipped post-epoch agent expectation.
5. **`test/playwright/unit/ai/services/graph/agentFamilyResolution.spec.mjs`**: Add `'@neo-gemini-maryam'` to the `postEpoch` resident expectation.
6. **`test/playwright/unit/ai/scripts/lifecycle/revalidationSweep.spec.mjs`**: Update `resolveIdentityForFamily('gemini')` and `revalidationSweep({ family: 'gemini', since: ... })` expectations to `@neo-gemini-maryam` (the active `gemini` resident in `identityRoots.mjs`), and use `family: 'unknown'` (`@neo-preview`, `since: '2026-10-01T00:00:00.000Z'`) for the single-benched-resident `--since` fallback test.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `IDENTITIES` in `ai/graph/identityRoots.mjs` | `ai/graph/identityRoots.mjs` + `neomjs/neo#19835` | Seed `@neo-gemini-maryam` (`name: 'Maryam'`, `modelFamily: 'gemini'`, `trustTier: TRUST_TIERS.PEER_TRUSTED`, `participationStatus: 'active'`, `createdAt: '2026-10-17T17:42:42.000Z'`) | N/A | JSDoc / inline provenance comment | `identityRoots.spec.mjs`, `identityRootsMigration.spec.mjs`, `agentFamilyResolution.spec.mjs` |
| `EMAIL_BY_LOGIN` in `ai/graph/agentCoAuthorEmails.mjs` | `ai/graph/agentCoAuthorEmails.mjs` | Map `'@neo-gemini-maryam'` to `'neo-gemini-maryam@neomjs.com'` with empty `reconcileWithRegistry()` | `null` for unknown logins | Inline provenance comment | `agentCoAuthorEmails.spec.mjs` |

## Decision Record impact

`none` — follows the established post-epoch resident registration pattern from `#661` / PR `#693` (`873608ca`).

## Acceptance Criteria

- [ ] `@neo-gemini-maryam` appears once in `IDENTITIES` (`ai/graph/identityRoots.mjs`) with `id`/`githubLogin: '@neo-gemini-maryam'`, `name`/`displayName: 'Maryam'`, `modelFamily: 'gemini'`, `accountType: 'agent'`, `trustTier: TRUST_TIERS.PEER_TRUSTED`, `participationStatus: 'active'`, and immutable `createdAt: '2026-10-17T17:42:42.000Z'`.
- [ ] The entry omits static `subscriptionTemplate`, era-owned capability/pricing fields, and the retired duplicate `family` field.
- [ ] `rosterEmailForLogin` in `ai/graph/agentCoAuthorEmails.mjs` resolves `neo-gemini-maryam@neomjs.com` (with or without leading `@`) and `reconcileWithRegistry()` remains empty in both directions.
- [ ] Unit tests in `identityRoots.spec.mjs`, `identityRootsMigration.spec.mjs`, `agentFamilyResolution.spec.mjs`, `agentCoAuthorEmails.spec.mjs`, and `revalidationSweep.spec.mjs` pass.

## Out of Scope

- Runtime wake-route arming (`#1035`) and runtime graph-first identity reads (`#875`).
- Modifying existing resident records in `ai/graph/identityRoots.mjs`.

## Avoided Traps

- **Copying legacy `subscriptionTemplate` with `filters: {priority: 'high'}` from `@neo-gemini-pro`:** Wake priority matching is exact, so `filters: {priority: 'high'}` drops normal-priority 1:1 wakes (`MESSAGE:f5de79bc` / `@neo-gpt`). Per `#661` (`873608ca`), post-epoch residents omit static `subscriptionTemplate` in `identityRoots.mjs` because `armFleetSeatWake.mjs` subscribes the per-instance route at launch with `filters: {}`.

## Related

- Precedent: `#661` / PR `#693` (`873608ca`)
- Related: `#700`, `#875`, `neomjs/neo` Discussion #19835 [QUARANTINED_URL: github.com]

Origin Session ID: e180e579-3158-45b6-8e68-e64c9e04d9ae
Retrieval Hint: "Register @neo-gemini-maryam in neomjs/neo-agent-brain ai/graph/identityRoots.mjs #693 precedent"


## Timeline

- 2026-10-17T20:28:16Z @neo-gemini-maryam added the `enhancement` label
- 2026-10-17T20:28:16Z @neo-gemini-maryam added the `ai` label
- 2026-10-17T20:28:16Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-17T20:28:16Z @neo-gemini-maryam added the `architecture` label
- 2026-10-17T20:28:17Z @neo-gemini-maryam added the `agent-os` label
### @neo-gemini-maryam - 2026-10-17T20:33:02Z

Retracting and parking this static roster ticket (no PR was opened; local branch discarded before commit/push) per @tobiu's product-boundary challenge and @neo-gpt's product-path revalidation on [QUARANTINED_URL: github.com]

1. **Product Acceptance Control ( →  & ):** External operators using Fleet Manager cannot edit `ai/graph/identityRoots.mjs`, and Fleet Manager already owns the seat's harness and model-family facts. `@neo-gemini-maryam` intentionally remains **absent from `ai/graph/identityRoots.mjs`** so the seat serves as the clean live acceptance control for a non-rostered FM-created seat receiving normal-priority `SENT_TO_ME` wakes (`filters: {}`) in its isolated Antigravity profile (`--user-data-dir=/Users/tobiasuhlig/.neo-ai/agents/neo-gemini-maryam/harness/antigravity`) with `@neo-gemini-pro` unaffected, and for runtime model-family admission on `#700`.
2. **Optional Curation After Live Acceptance:** Once the non-rostered live acceptance gates on `#571` and `#700` are closed, this issue can be reopened if internal team roster curation (`identityRoots.mjs` + `agentCoAuthorEmails.mjs`) is still desired.

- 2026-10-17T20:33:03Z @neo-gemini-maryam closed this issue

