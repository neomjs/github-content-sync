---
id: 446
title: 'A PAT the plane refuses while the shell runs gets Connect, not Reconnect'
state: OPEN
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-02T13:34:28Z'
updatedAt: '2026-10-02T14:35:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/446'
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
# A PAT the plane refuses while the shell runs gets Connect, not Reconnect

## Context

FM v1 ROADMAP row 5 (epic #424) needs "the PAT expires or is wrong" to return to `live` by the product's own guidance. Clio's row-5 script expects "a named credential verdict in the banner or the plane chip, and the connect card to re-enter" the PAT (#335, step 5). Leaf 1 (#425) typed that refusal at boot. This leaf is the runtime half the epic names: while the shell runs, the banner's only action is Reconnect, which cannot fix a credential.

Measured 2026-10-02 (Brain `dev@a9dd22f`, Institution `dev@7cd284b`):

1. **The plane tells the two failures apart.** Given a bearer it does not accept, the local plane answers HTTP 401. A port with no plane behind it refuses the connection.
2. **The Fleet's plane client does not.** With that bearer, `createPlaneMailboxClient` reports the following, with no code and no `blockerCode` on either:
   - at init: `{"ok":false,"reason":"plane unreachable (Error)"}`
   - on a call: `plane list_messages failed: session lost and plane unreachable (Error)`

   The MCP SDK's `StreamableHTTPError` carries `code: 401`, but the client keeps only the error's name. Vega saw the same collapse on 2026-09-25: "plane unreachable" was an admission failure wearing a connectivity name.
3. **The wire carries no cause.** `dispatchFleetRequest` turns every thrown read into `operation-failed` with a sanitized message.
4. **The cockpit cannot tell either.**
   - `installFleetBridge` maps `operation-failed` to `failed-upstream`.
   - `SpineBanner.connectionVerdict` returns `action: null` for every connection state.
   - `cockpit/Container.mjs` renders a null action as Reconnect.

So when a PAT expires or is revoked mid-session, the cockpit says "fleet failed · Roster request reported an upstream failure" and offers Reconnect, which sends the same refused PAT again.

## The Problem

The renderer cannot tell a refused credential from any other failure, and by design it should not guess: `connectionVerdict`'s copy "names the request, never an authentication decision". Only the shell holds both the saved plane record and the probe that tells the causes apart, and today it runs that probe only at boot.

## The Architectural Reality

- **The probe.** `harness/planeConfig.mjs` `probePlaneCredential({planeBase, bearer})` answers `accepted | rejected | not-a-plane | no-identity | unreachable`, plus the identity on acceptance. The attach handler beside it prompts for a PAT, probes it, writes the plane record and relaunches. That is what Connect leads to.
- **Boot typing.** `harness/brain.mjs` `typePlaneRefusal` runs the probe at boot and types a refused attach. The causes are `plane-credential-refused`, `plane-identity-changed`, `plane-not-a-plane` and `plane-unreachable`.
- **The banner.** `apps/agentos/util/SpineBanner.mjs` `PLANE_REFUSALS` maps those causes, read from the daemon surface, to the connect card's words with action `connect-plane`. `cockpit/Container.mjs` renders that action as Connect, and `cockpit/Controller.mjs` `onSpineAction` opens the plane card for it; every other verdict reconnects the fleet.
- **The runtime failure.** A failed roster read reaches `cockpit/LivenessController.mjs` as `fleetConnectionState` (`refused | unreachable | timeout | failed-upstream`). Nothing asks the plane why.

## The Fix

1. **The shell can re-ask the plane.** The plane broker gains a third named capability beside `planeStatus()` and `attachPlane()`: `verifyPlane()` → `ipcRenderer.invoke('shell-plane-verify')`. Its main-process handler validates the sender, probes the record this shell launched its fleet child with (`launchedPlaneRecord`, the rule leaf 1 uses at boot) and answers `{cause}`. It never answers the PAT, and it never probes a plane that the environment supplied instead of the record.
2. **The cockpit asks when a roster read fails at runtime.** On a `failed-upstream` or `refused` roster read, with a plane record saved, the liveness owner asks once per failure episode. A read that recovers ends the episode. It never polls while the failure stays.
3. **Credential verdicts become the causes the banner already speaks.**
   - `rejected` becomes `plane-credential-refused`.
   - An accepted PAT that names another identity becomes `plane-identity-changed`.
   - Both show the existing `PLANE_REFUSALS` words with Connect.
   - `unreachable` and `not-a-plane` keep today's Reconnect. A plane that stops answering mid-session is restarting, which is its own row-5 step, and re-entering a PAT cannot help it.
4. **Recovery clears the cause.** A Connect that re-attaches, or a later successful read, clears the runtime cause.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `verifyPlane()` preload capability + `shell-plane-verify` handler (new) | ADR 0034 §2.3 item 8, amended first; `probePlaneCredential` | Probes the launched plane record and answers `{cause}`: `plane-credential-refused`, `plane-identity-changed`, or `null` | No launched record, or a plane from the environment: `{cause: null}`. A probe that throws reads as no cause. The PAT never crosses IPC | ADR 0034 §2.3 item 8, JSDoc | AC-1 |
| `LivenessController` runtime failure path (existing) | this ticket | One verify per failure episode on a `failed-upstream` or `refused` roster read; only the asking episode's answer is published | No launched record (own mode, dev transport): the cockpit still asks, and the broker answers `{cause: null}` without probing. The renderer never learns whether a record exists | JSDoc | AC-1, AC-2 |
| `SpineBanner` plane causes (existing `PLANE_REFUSALS`) | leaf 1's vocabulary | At runtime, `plane-credential-refused` and `plane-identity-changed` show Connect | `unreachable` and `not-a-plane` keep the Reconnect path | JSDoc | AC-3 |

## Decision Record impact

`amends ADR 0034` (§2.3 item 8, in `neomjs/neo`): the plane-attach broker pair names exactly two capabilities, so a third, `verifyPlane()`, is an amendment. It lands first as neomjs/neo#19362, before this leaf's PR, the way the pair itself did (neomjs/neo#19228). ADR successor-risk: `adr-amendment-required`.

## Acceptance Criteria

- [ ] AC-1 (unit): the verify handler answers `plane-credential-refused` for a launched PAT the plane answers with 401, `plane-identity-changed` for one it admits as another account, and `null` for an unchanged PAT, an unreachable plane, or no launched record. It refuses an untrusted sender and never answers the PAT. The preload census in `main.mjs` names the third capability.
- [ ] AC-2 (unit): a runtime `failed-upstream` roster read asks once per failure episode. A recovered read followed by a second failure asks again, and only the asking episode's answer is published, whichever episode's reply lands last. With no launched record the broker answers `{cause: null}` without probing the plane (AC-1's never-probed arm).
- [ ] AC-3 (unit): a `rejected` verdict renders the banner's PAT-refused words with Connect, and the next successful read clears it. An `unreachable` verdict keeps Reconnect.

## Post-Merge Validation

- [ ] In epic #424's installed sitting, step 5: revoke the PAT while the shell runs. The banner names the refused PAT and offers Connect, and Connect with a valid PAT returns to `live`.

## Out of Scope

- **Typing the refusal on the Brain wire.** The plane client and `dispatchFleetRequest` keep their sanitized failure; the shell asks the plane itself, as the epic prescribes.
- **The plane-restart and stale-plane steps** of row 5.
- **The boot-time refusal**, which is leaf 1's (#425).

## Avoided Traps

- **A typed cause on the Brain wire:** three layers in two repositories for a fact the shell can ask the plane directly. `dispatchFleetRequest` also deliberately never exposes a raw error.
- **Promoting runtime `unreachable` to Connect:** a restarting plane would send the operator to re-enter a PAT.
- **Polling the plane while degraded:** one ask per failure episode, never a loop.
- **Parsing the fleet child's lines:** the epic rules it out. The shell types a failure with its own probe.

## Related

#424 (parent) · #425 (leaf 1, boot typing) · #335 (the row-5 script and audit) · #15 (the banner's remote states, which carry no plane-owned failure)

Live latest-open sweep: the latest 20 open Institution issues at 2026-10-02T13:32:42Z, no equivalent. #15 lists an `auth-refused` banner state, which the 09:08Z agreement with its author assigned to #424's leaves; runtime is this leaf. A2A in-flight sweep (all read states, last 30 messages): no claim on this scope. MC sweep ("PAT expired revoked while the cockpit runs, banner only offers Reconnect, credential refused at runtime, fleet failed upstream"): 6 results, no prior decision; Vega's 2026-09-25 receipt shows the same 401-as-unreachable collapse. Own-assignment sweep: 1 open (#424, the parent). KB ticket sweep: #15, #335, #225 (closed, the boot refusal that offers Connect), neomjs/neo#16699 (closed, the Reconnect affordance).

Origin Session ID: 6f7d14a3-e126-4b47-888f-fc28c748ae83
Retrieval Hint: "runtime PAT refused banner Reconnect Connect plane-verify probePlaneCredential row 5 step 5"

⚖️ Ada (Claude Opus 5.5, Claude Code)



## Timeline

- 2026-10-02T13:34:29Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-02T13:34:30Z @neo-opus-ada added the `enhancement` label
- 2026-10-02T13:34:30Z @neo-opus-ada added the `ai` label
- 2026-10-02T13:34:38Z @neo-opus-ada added parent issue #424
- 2026-10-02T13:40:15Z @neo-opus-ada cross-referenced by #19362
- 2026-10-02T13:42:00Z @neo-opus-ada cross-referenced by PR #19363
- 2026-10-02T13:55:38Z @neo-opus-ada cross-referenced by PR #447
- 2026-10-02T14:36:53Z @neo-opus-ada referenced in commit `5ab1fe7` - "fix(agentos): a plane-credential answer belongs to the episode that asked for it (#446)

PlaneCredentialCheck kept one Boolean per failure episode. After a read
recovered and a second failure opened a new episode, the first episode's
late answer passed the check and overwrote the newer one, in either
direction.

Each episode now asks with its own token. settle() clears the token, and an
answer publishes only while its own token is current. One ask per episode
stays.

The spec adds the control for recovery, then a second failure, then the
newer reply, then the older reply, in both result directions, beside the
existing single-recovery control."
- 2026-10-02T14:45:55Z @neo-opus-ada referenced in commit `08421a4` - "test(visual): the golden stamp names the episode-token inputs, re-rendered with no pixel change (#446)"
- 2026-10-02T15:29:38Z @neo-opus-ada cross-referenced by #424

