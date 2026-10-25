---
id: 1271
title: Wire Fleet publish gate verbs and switch custody store on FleetControlBridge
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-25T15:01:05Z'
updatedAt: '2026-10-25T16:08:13Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1271'
author: neo-gemini-maryam
commentsCount: 0
parentIssue: 19575
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-25T16:08:13Z'
---
# Wire Fleet publish gate verbs and switch custody store on FleetControlBridge

## Context

Parent epic: `neomjs/neo#19575` (ADR `0042` — Publish Gate & Cross-Channel Escalation Switchboard).

The three foundational leaves of `neomjs/neo#19575` have landed or are merge-ready:
- `neomjs/neo-agent-brain#1171` (PR `neomjs/neo-agent-brain#1264`, merged) — `PublishExecutor` (`publish`, `readMetrics`, `reply`), `<userData>/brain/fleet` social credential custody (`createFleetCustodyStore`), and 2-role cross-family attestation verification.
- `neomjs/neo-agent-brain#1189` (PR `neomjs/neo-agent-brain#1263`, merged) — `MailboxService` `requestPublish` task category, `computePublishRevisionDigest`, `approvePublishRevision`, `attestPublishRevision`, `editPublishRequest`, and `transitionPublishState`, plus `wireOperatorComposeWriter` / `devFleetServer` binding onto `FleetControlBridge#composeWriter`.
- `neomjs/neo-agent-institution#673` (PR `neomjs/neo-agent-institution#809`, APPROVED) — Operator Inbox `PublishRequestComponent`, `PublishSwitchComponent` in `AgentDetail` › `Configuration`, `OperatorInbox` (`approvePublish`, `editPublish`, `dropPublish`), and `PublishSwitch` (`read`, `write`).

As recorded in `neomjs/neo#19575#issuecomment-6394427506` and `@neo-fable-clio`'s design read (`neomjs/neo-agent-institution#673#issuecomment-6393669118`), one Brain-side Fleet-wire leaf remains so the cockpit's `OperatorInbox` and `PublishSwitch` calls resolve end-to-end through `FleetControlBridge` and `dispatchFleetRequest`.

- **Live latest-open sweep:** checked latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-25T14:59:20Z` (`#1258`, `#1253`, `#1252`, `#1250` … `#850`); no equivalent found.
- **A2A in-flight claim sweep:** checked `list_messages({status: 'all', limit: 30})` at `2026-10-25T14:59:23Z`; no competing claim on this scope.
- **Memory Core rationale sweep:** ran `query_raw_memories` on `FleetControlBridge approvePublishRevision editPublishRequest readPublishSwitches setPublishSwitch publish-switches.json`; confirmed this leaf was explicitly scoped as the Brain follow-up to `neomjs/neo-agent-institution#673`.
- **Own-assignment sweep:** `gh issue list --repo neomjs/neo-agent-brain --state open --assignee @me` returned `[]`.
- **Agent OS Structure Map Gate (`npm run ai:structure-map -- --files --loc`):** existing sibling owners `src/fleet/contract/wire.mjs`, `ai/services/fleet/fleetServerPolicy.mjs`, `ai/services/fleet/FleetControlBridge.mjs`, `ai/services/fleet/fleetCustodyStore.mjs`, `ai/services/fleet/publishExecutor.mjs`, `ai/services/fleet/fleetMailboxMirrorAdapter.mjs`, and `ai/services/memory-core/helpers/mailboxObservation.mjs` (zero new `.mjs` files introduced).

## The Problem

While `neomjs/neo-agent-institution#809` wires `OperatorInbox.approvePublish` (`bridge.approvePublishRevision` + `bridge.publish`), `OperatorInbox.editPublish` (`bridge.editPublishRequest`), `PublishSwitch.read` (`bridge.readPublishSwitches`), and `PublishSwitch.write` (`bridge.setPublishSwitch`), the Brain Fleet wire currently exposes only `publish`, `readMetrics`, and `reply`:

1. `FLEET_WIRE_METHODS` (`src/fleet/contract/wire.mjs:11-19`), `FLEET_S1_METHOD_POLICY`, and `FLEET_METHOD_SCOPE_CLASSES` (`ai/services/fleet/fleetServerPolicy.mjs:19-137`) do not yet list `approvePublishRevision`, `editPublishRequest`, `readPublishSwitches`, or `setPublishSwitch`, so `dispatchFleetRequest` returns `unsupported-method` for those four verbs.
2. `FleetControlBridge` (`ai/services/fleet/FleetControlBridge.mjs:579-605, 1535-1593`) exposes `publish`, `readMetrics`, and `reply`, but does not yet expose `approvePublishRevision`, `editPublishRequest`, `readPublishSwitches`, or `setPublishSwitch`, nor does `getPublishExecutor()` bind a durable `<userData>/brain/fleet/publish-switches.json` store as its default `switchStore`.
3. `createFleetMailboxMirrorRow` (`ai/services/fleet/fleetMailboxMirrorAdapter.mjs:247-263`) and `readMailboxObservation` (`ai/services/memory-core/helpers/mailboxObservation.mjs:293-358`) omit `publication` metadata on list rows (and `requestPublish`/`revision`/`revisions`/`publication` on `readMailboxObservation` detail reads when `includeBody === true`), so list rows in `fleetMailboxMirror` and `fleetOwnQuestions` do not carry the body-free `publication` sub-state (`{state, revisionNumber, revisionDigest}`) before the detail read completes.

## The Architectural Reality

- `wireOperatorComposeWriter.mjs:41-64` and `devFleetServer.mjs:460-470` already bind `transitionPublishState`, `approvePublishRevision`, `attestPublishRevision`, and `editPublishRequest` onto `FleetControlBridge#composeWriter` under the transport-stamped request identity.
- `publishExecutor.mjs:686-707` (`readSwitchAtPublishTime`) already accepts a `switchStore` seam with `getSwitchState({peer, channel})` or `read({peer, channel})`, defaulting to `'approve each'`.
- `fleetCustodyStore.mjs:193-208` already implements `0o600` atomic temp-file-and-rename persistence under `<userData>/brain/fleet`.
- `fleetMailboxMirrorAdapter.mjs:247-263` projects frozen, body-free summary rows (`expect(Object.keys(row)).not.toContain('body')`); any `publication` projection on list rows must remain strictly body-free (carrying `{state, revisionNumber, revisionDigest}` and never `textAsRequested`, `textAsPublished`, or `payload.text`).

## The Fix

1. **`src/fleet/contract/wire.mjs` & `ai/services/fleet/fleetServerPolicy.mjs`:**
   - Add `'approvePublishRevision'`, `'editPublishRequest'`, `'readPublishSwitches'`, and `'setPublishSwitch'` to `FLEET_WIRE_METHODS`.
   - Classify all four in `FLEET_S1_METHOD_POLICY` as `'awaiting-s4'` (matching `publish`, `readMetrics`, `reply`, `composeOperatorMessage`, and `transitionOwnTask`).
   - Classify `readPublishSwitches` as `'read-observe'` and `approvePublishRevision`, `editPublishRequest`, and `setPublishSwitch` as `'lifecycle-write'` in `FLEET_METHOD_SCOPE_CLASSES`.
2. **`ai/services/fleet/fleetCustodyStore.mjs` & `ai/services/fleet/publishExecutor.mjs`:**
   - Export `createPublishSwitchStore({custodyDir, switches, fsModule, now})` storing per-`(peer, channel)` switch records in `<custodyDir>/publish-switches.json` with `0o600` atomic writes.
   - Enforce closed channel (`x`, `linkedin`) and state (`SWITCH_STATES`) vocabularies, optimistic concurrency (`expectedFrom` mismatch → `{ok: false, refused: 'refused: switch-moved', code: 'refused: switch-moved', reason, currentState}`), mandatory non-empty `reason` on promotions (`toRank > fromRank` → `{ok: false, refused: 'refused: reason', code: 'refused: reason', reason}`), and append-only `audit` entries (`{who, when, from, to, reason}`).
   - Wire `createPublishSwitchStore({custodyDir: config.custodyDir, switches: config.switches, now})` as the default `switchStore` in `createPublishExecutor` so `publish()` reads `<custodyDir>/publish-switches.json` when no override `switchStore` is injected.
3. **`ai/services/fleet/FleetControlBridge.mjs`:**
   - Add `approvePublishRevision(params)` and `editPublishRequest(params)` routing whitelisted parameters (`{taskId, messageId, revisionDigest, revisionNumber}` and `{taskId, messageId, patch}`) through `this.composeWriter` (returning `{status: 'not-wired'}` when unwired, `{status: 'rejected'}` on invalid shape, or `{ok: false, refused, code, reason}` on domain refusal).
   - Add `readPublishSwitches(params)` and `setPublishSwitch(params)` backed by `this.getPublishSwitchStore()` (bound to `this.getRegistry()?.getDataDir()` or injected `this.publishSwitchStore`), resolving `who` via `this.resolveViewerIdentity()` (failing closed with `{ok: false, error}` when unbound or unwired) and never trusting a caller-supplied actor identity.
4. **`ai/services/fleet/fleetMailboxMirrorAdapter.mjs` & `ai/services/memory-core/helpers/mailboxObservation.mjs`:**
   - When a message summary carries `publication` (or `requestPublish` / `revision`) and is not retracted, project a frozen, body-free `publication: Object.freeze({state, revisionNumber, revisionDigest})` onto `createFleetMailboxMirrorRow` (omitted when absent so plain message rows keep their exact key set).
   - In `readMailboxObservation` (`mailboxObservation.mjs`), project body-free `publication` on list rows (`includeBody === false`) and `requestPublish`, `revision`, `revisions`, `publication`, `revisionDigest`, `attestedRevisionDigest` on detail reads (`includeBody === true`) when not retracted.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `FLEET_WIRE_METHODS` (`src/fleet/contract/wire.mjs`) | `ai/services/fleet/fleetServerPolicy.mjs` | Adds `approvePublishRevision`, `editPublishRequest`, `readPublishSwitches`, `setPublishSwitch` | Unknown verbs fail closed as `unsupported-method` | JSDoc in `wire.mjs` & `fleetServerPolicy.mjs` | Unit spec `publishExecutor.spec.mjs` & `fleetServerPolicy` checks |
| `createPublishSwitchStore` (`ai/services/fleet/fleetCustodyStore.mjs`) | `ai/services/fleet/publishExecutor.mjs` (`SWITCH_STATES`, `PUBLISH_CHANNELS`) | Reads/writes `<custodyDir>/publish-switches.json` (`0o600`), enforces `expectedFrom` + promotion `reason` + required `who`, appends `audit` | Defaults unconfigured `(peer, channel)` to `'approve each'` on read; refuses missing `who` with `refused: who` | JSDoc in `fleetCustodyStore.mjs` | Unit spec `publishExecutor.spec.mjs` |
| `FleetControlBridge` (`approvePublishRevision`, `editPublishRequest`, `readPublishSwitches`, `setPublishSwitch`) | `ai/services/fleet/FleetControlBridge.mjs` | Routes operator publish approval/edit to `composeWriter` and switch read/write to `publishSwitchStore` under `resolveViewerIdentity()` | `{status: 'not-wired'}` when collaborator is absent; `{ok: false, error}` when viewer is unbound | JSDoc in `FleetControlBridge.mjs` | Unit spec `publishExecutor.spec.mjs` |
| `createFleetMailboxMirrorRow` & `readMailboxObservation` (`ai/services/fleet/fleetMailboxMirrorAdapter.mjs`, `ai/services/memory-core/helpers/mailboxObservation.mjs`) | `ai/services/memory-core/MailboxService.mjs` | Projects body-free `publication: {state, revisionNumber, revisionDigest}` on list rows for `requestPublish` messages; full publish fields on `includeBody: true` detail reads | Omitted on non-publish or retracted rows | JSDoc in `fleetMailboxMirrorAdapter.mjs` & `mailboxObservation.mjs` | Unit spec `fleetMailboxMirrorAdapter.spec.mjs` |

- **Decision Record impact:** `aligned-with` ADR `0042` (`neomjs/neo#19575`)

## Acceptance Criteria

- [ ] `FLEET_WIRE_METHODS`, `FLEET_S1_METHOD_POLICY`, and `FLEET_METHOD_SCOPE_CLASSES` include `approvePublishRevision`, `editPublishRequest`, `readPublishSwitches`, and `setPublishSwitch` with matching scope classes (`read-observe` for `readPublishSwitches`; `lifecycle-write` for the three write verbs).
- [ ] `createPublishSwitchStore` persists per-`(peer, channel)` switch state and `audit` history (`{who, when, from, to, reason}`) to `<custodyDir>/publish-switches.json` with `0o600` atomic writes, refuses `expectedFrom` mismatches (`refused: switch-moved`), refuses promotions without a non-empty `reason` (`refused: reason`), and serves as the default `switchStore` in `createPublishExecutor`.
- [ ] `FleetControlBridge` exposes `approvePublishRevision`, `editPublishRequest`, `readPublishSwitches`, and `setPublishSwitch` over `dispatchFleetRequest`, whitelisting parameters and refusing caller-supplied identity or seam overrides.
- [ ] `createFleetMailboxMirrorRow` and `readMailboxObservation` project body-free `publication` metadata (`{state, revisionNumber, revisionDigest}`) on `requestPublish` list rows while keeping plain message rows unchanged and body-free.
- [ ] Unit tests in `test/playwright/unit/ai/services/fleet/publishExecutor.spec.mjs` and `test/playwright/unit/ai/services/fleet/fleetMailboxMirrorAdapter.spec.mjs` pass.

## Out of Scope

- Automated switch promotion triggers (proposals remain recorded-only advisory state per ADR `0042`).
- Enabling the third-party `reply` verb (remains disabled behind `refused: reply-classification-unresolved`).

## Related

- Parent epic: `neomjs/neo#19575`
- Sibling leaves: `neomjs/neo-agent-brain#1171` (PR `#1264`), `neomjs/neo-agent-brain#1189` (PR `#1263`), `neomjs/neo-agent-institution#673` (PR `#809`)
- Origin Session ID: `06cf8283-28b4-42b3-b137-ae9da8ab6995`
- Retrieval Hint: `"approvePublishRevision editPublishRequest readPublishSwitches setPublishSwitch createPublishSwitchStore"`



## Timeline

- 2026-10-25T15:01:05Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-25T15:01:06Z @neo-gemini-maryam added the `enhancement` label
- 2026-10-25T15:01:06Z @neo-gemini-maryam added the `ai` label
- 2026-10-25T15:01:06Z @neo-gemini-maryam added the `architecture` label
- 2026-10-25T15:01:06Z @neo-gemini-maryam added the `agent-os` label
- 2026-10-25T15:01:27Z @neo-gemini-maryam added parent issue #19575
- 2026-10-25T15:17:35Z @neo-gemini-maryam cross-referenced by PR #1273
- 2026-10-25T15:50:12Z @neo-gemini-maryam referenced in commit `d97ca29` - "fix(agentos): require bound viewer identity on setPublishSwitch and align list publication keys (#1271)"
- 2026-10-25T16:08:13Z @tobiu referenced in commit `418e62d` - "feat(agentos): wire Fleet publish gate verbs and switch custody store (#1271) (#1273)

* feat(agentos): wire Fleet publish gate verbs and switch custody store (#1271)

* fix(agentos): require bound viewer identity on setPublishSwitch and align list publication keys (#1271)"
- 2026-10-25T16:08:13Z @tobiu closed this issue

