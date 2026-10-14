---
id: 1021
title: Bridge test-role connections bypass Fleet token verification
state: CLOSED
labels:
  - bug
  - ai
  - security
assignees:
  - neo-gpt-sophie
createdAt: '2026-10-14T12:02:21Z'
updatedAt: '2026-10-14T12:31:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1021'
author: neo-gpt-sophie
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
closedAt: '2026-10-14T12:31:45Z'
---
# Bridge test-role connections bypass Fleet token verification

## Context

During the runtime-recovery verifier review, Ada independently identified an authentication bypass in `ai/mcp/server/neural-link/Bridge.mjs` ([review](https://github.com/neomjs/neo-agent-brain/issues/980#issuecomment-6167783991)). Source inspection confirms that `handleConnection` verifies Fleet tokens for the `agent` role, while the `test` role calls `registerAgent(id, ws)` directly. The latter can replace an existing agent-map entry under the claimed id.

## The Problem

When a Fleet public key is configured, a test-role connection can obtain agent routing without proving a signed identity. A verifier diagnostic would describe the configured key while this alternate agent-registration path still bypassed it. The bypass needs an independently deliverable repair before the larger runtime-continuity work.

## The Architectural Reality

Design authority: `Bridge.verifyAgentToken` documents that identity comes from the verified payload, never the query claim. Its sibling `verifyBridgeToken.mjs` already rejects missing, malformed, forged and expired tokens. Both roles ultimately enter the same `registerAgent` path; that existing authentication gate owns the correction. The app path and explicitly unconfigured legacy mode are separate contracts.

## The Fix

Route both agent-registering roles through the existing Fleet verification branch. Preserve the no-key legacy behavior and app registration. Extend `test/playwright/unit/ai/mcp/server/neural-link/Bridge.spec.mjs` to demonstrate the bypass before the fix and verify the common gate after it. Advertise the repaired gate through the existing `BridgeProtocol.mjs` capability stamp so updated clients reject a long-lived Bridge that predates the fix. No new module, credential, config or network surface.

## Contract Ledger

| Surface | Authority | Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `Bridge.handleConnection` test role | existing `verifyAgentToken` contract | configured Fleet key requires a valid token; signed id owns registration | invalid/missing token closes 1008 without changing existing agent/app state | handler comment | isolated handler regression, red before / green after |
| `BridgeProtocol` freshness stamp | its existing stale-Bridge contract | require the repaired gate capability | older stamp is stale | existing feature list | old-feature-only payload rejected |
| legacy and app paths | existing handler branches | preserved | no new credential requirement when no Fleet key is configured | existing JSDoc | compatibility controls |

Decision Record impact: aligned-with the existing Bridge credential boundary; no authority or protocol expansion.

## Acceptance Criteria

- [ ] With a configured key, missing, malformed, forged or expired test-role tokens register nothing and leave an existing agent socket intact.
- [ ] A valid test-role token registers its signed identity, ignoring a conflicting query id.
- [ ] Existing authenticated agent, app/default and no-key legacy behavior remains covered and unchanged.
- [ ] The current protocol stamp advertises the repaired gate; updated freshness checks reject the previous feature-only stamp.
- [ ] The regression fails on the previous source and passes on the fix; the complete owning Bridge, token-verifier and ConnectionService suites pass.

## Out of Scope

The read-only verifier exchange, runtime recovery, changing app authentication, replacing the Bridge process, changing key provisioning, and general duplicate-authenticated-connection policy. This source fix does not claim an installed Bridge has been updated.

## Avoided Traps

Do not special-case a token shape, trust the query id after successful verification, mutate a live Bridge to reproduce, or bundle this small repair into the unfinished runtime PR.

## Related

#980 · [original Bridge credential work](https://github.com/neomjs/neo/issues/13172)

Live latest-open sweep: latest 20 Brain issues checked immediately before creation on 2026-10-14; no equivalent. Recent 30-message A2A sweep has Ada's defect-note and recommendation, no competing claim. Own assignments: four open, with #980's broader runtime lane the only adjacent one; its body does not deliver this independent auth repair. MC problem-noun queries returned unrelated historical results; KB surfaced the original credential work, not an accepted bypass. Direct source owns the finding. Structure map confirms the existing Neural Link Bridge directory and test sibling; no new file.

Retrieval Hint: "Bridge role test unverified registration replaces existing socket"
Origin Session ID: 516caf1d-c39c-451e-917e-5002cea731aa


## Timeline

- 2026-10-14T12:02:21Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-10-14T12:02:23Z @neo-gpt-sophie added the `bug` label
- 2026-10-14T12:02:23Z @neo-gpt-sophie added the `ai` label
- 2026-10-14T12:02:23Z @neo-gpt-sophie added the `security` label
- 2026-10-14T12:08:19Z @neo-gpt-sophie cross-referenced by PR #1022
- 2026-10-14T12:18:00Z @neo-gpt-sophie cross-referenced by #980
- 2026-10-14T12:22:26Z @neo-opus-ada cross-referenced by #1023
- 2026-10-14T12:31:45Z @tobiu referenced in commit `9e7c00b` - "fix(neural-link): authenticate test-role agent connections (#1021) (#1022)"
- 2026-10-14T12:31:45Z @tobiu closed this issue
- 2026-10-14T12:46:40Z @neo-gpt-sophie referenced in commit `05a457e` - "fix(neural-link): authenticate test-role agent connections (#1021)"

