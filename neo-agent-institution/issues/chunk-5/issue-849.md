---
id: 849
title: A boot refusal that is not the credential's needs a new PAT to recover
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-31T00:05:12Z'
updatedAt: '2026-10-31T00:05:13Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/849'
author: neo-opus-ada
commentsCount: 0
parentIssue: 424
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
# A boot refusal that is not the credential's needs a new PAT to recover

## Context

Operator, 2026-10-30 (prio 0, the incident in #847): after the FM's login wait ended on "not a Neo plane", the operator could not get back without generating a new PAT. The stored PAT was valid: nothing had revoked it, and the plane admits PATs the same way minutes later. #847 stops the wait from ending while the plane starts. This leaf is the way back from the refusals that can still end a boot.

## The Problem

Every boot refusal's lead points at Connect:

- "Connect to the plane's own address."
- "Bring that plane back, or connect to another."
- "The plane refused this shell — connect it again"

(`SpineBanner` `PLANE_REFUSALS` and `coldFallbackFor`, each with the `connect-plane` action.) Connect always prompts for a PAT (`createPlaneBroker().attach` → `promptCredential`), and the shell never retries its stored record once a boot has settled.

So any refusal that is not about the credential costs a PAT the operator may not have at hand. Examples: a probe that timed out (`plane-refused`), a plane that answered as another server, a Fleet child that exited for its own reason. Rotating the PAT also leaves the old one live.

## The Architectural Reality

- `harness/planeConfig.mjs` `createPlaneBroker` offers `status`, `attach` (prompt → probe → write → relaunch) and `verify`. Nothing retries with the stored record.
- `harness/main.mjs` runs `bootProductBrain()` once. Its `.catch` settles the boot down (`appLifecycle.settleBrainBoot(false, cause)`).
- `apps/agentos/util/SpineBanner.mjs` `PLANE_REFUSALS` gives every refusal the `connect-plane` action.
- #843 (open) covers the other half of the cost: Connect's relaunch. This leaf must not add a relaunch.

Design authority: #425 Avoided Traps — "Re-probing on a timer. One probe per failed boot is enough. Recovery is the operator's Connect." That ruling assumed Connect costs nothing. It costs a PAT.

## The Fix

1. Add a `Retry` action to every boot refusal except the credential's own two (`plane-credential-refused`, `plane-identity-changed`).
2. Retry makes main re-run the plane-attach boot with the stored record, in-process and with no `app.relaunch()`, through a trusted IPC channel beside `shell-plane-*`.
3. The banner returns to `starting` while the retry runs.
4. Connect stays for a different plane or a different PAT.

## Acceptance Criteria

- [ ] AC-1: a boot that settled with a non-credential refusal and has a stored record recovers through Retry, with no PAT prompt and no relaunch. Unit test with broker and boot seams.
- [ ] AC-2: the credential's own two refusals offer Connect only. Spine banner spec.
- [ ] AC-3: Retry never carries the PAT to the renderer. Spec.
- [ ] AC-4 `[L4-deferred]`: on the installed FM, a boot refusal followed by Retry reaches `live`. Receipt on #516.

## Out of Scope

- A cold Memory Core that cannot reach GitHub answering a valid PAT with 401 (#847, Out of Scope).
- Adopting a new PAT without a relaunch (#843).

## Related

Part of #424 · #847 · #843 · #425 · promotes the defect-note "the FM cockpit's boot-refusal recovery is wrong …" (2026-10-30T23:49Z) on the operator's escalation.

Live latest-open + closed sweep: the latest 20 open issues and closed searches ran at 2026-10-30T23:47Z for #847 and were re-checked at the newest open issues just before filing. #425 and #446 were read (#446: a PAT refused mid-session gets Connect). No equivalent. A2A sweep: no claim on this scope. Own-assignment sweep: #843 is adjacent (new-PAT relaunch), not equivalent. Structure map: N/A (existing `harness/` and `apps/agentos/`). Decision Record impact: none.

Retrieval Hint: "FM boot refusal retry stored PAT Connect demands new PAT rotated"


## Timeline

- 2026-10-31T00:05:13Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-31T00:05:14Z @neo-opus-ada added the `enhancement` label
- 2026-10-31T00:05:14Z @neo-opus-ada added the `agent-os` label
- 2026-10-31T00:05:14Z @neo-opus-ada added the `ai` label
- 2026-10-31T00:05:15Z @neo-opus-ada added parent issue #424

