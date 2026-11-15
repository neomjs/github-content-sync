---
id: 1731
title: Fleet provisions native wakes for managed Codex and Antigravity seats
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-11-15T00:57:32Z'
updatedAt: '2026-11-15T03:09:38Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1731'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 1735 Retire AppleScript message delivery after native peer validation'
closedAt: '2026-11-15T03:09:38Z'
---
# Fleet provisions native wakes for managed Codex and Antigravity seats

## Context

Design authority: the operator wants another operator to Add and Start a managed Codex Desktop or Antigravity App seat and receive A2A in its current main chat without editing a harness profile, running enrollment commands, or setting up each peer by hand. [Native transport and ownership contract](https://github.com/neomjs/neo-agent-brain/issues/1706) · [installed trial](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6072691768)

## The Problem

The signed transport works in the Codex trial. Ordinary Start still reads a manually created per-seat enrollment, and without it `armFleetSeatWake` selects window typing for both families. The Antigravity sidecar and workspace-hook writers and the Codex seat-hook writer exist, but normal Start does not call them. Publishing the subscription does not establish that any chat can receive a wake.

## The Architectural Reality

`FleetManager.startAgent` owns per-seat serialization and Stop cancellation. Its provisioning collaborator owns prelaunch preparation. `armSeatWake` binds the admitted process and generation after launch. The receiver, enrollment, spool and recipient hook already fence replay, generations, supersession and subagents. Codex’s verifier currently accepts exactly the seat-home hook, its command and hashes and an ordinary invocation in that process. The Gemini App sidecar setting is shared by the OS user, while the two hooks live in the registered project. The existing Fleet Wake Routes read distinguishes publication and delivery.

## The Fix

Use these production boundaries to make native delivery the default on the two supported desktops. Prepare the files before the normal launch, bind from the actual host receiver declaration, then bind the live process. Cover an already-running managed seat without terminating it. Make retries converge after a partially completed setup and preserve unrelated hooks, explicitly disabled entries and existing signed records. Keep native requests pending during a missing trust, hook, root or receipt; migrate old GUI records without falling back to typing while a native setup is incomplete. Use stable, ownership-verifiable hook commands across supported updates and remove only Fleet-owned registrations on release. A first installation needs a supported receiver setup or an actionable missing-receiver state.

Rollout order (operator, November 15): this ticket ships normal Fleet preparation, a guarded setup action for the current running process, and native routing for the seats selected in our real-peer trial. Ordinary Start may stage private files during the trial. It keeps existing delivery until a seat is selected. Once real Codex and Antigravity peers prove signed receipt in the right main chat, #1735 makes native the only ordinary route and removes AppleScript message senders. No permanent compatibility setting. The separate Claude sunset operation remains available. OpenCode is a separate decision.

Codex setup is strictly seat-local. Changes to system policy, MDM, cloud-managed configuration or any hook loaded in a private Codex instance are out of scope. Use the seat’s isolated `CODEX_HOME`, stage the exact Fleet hook, and make any vendor-required approval clear inside FM. Never copy another seat’s trust record, synthesize a trusted hash or bypass hook review. Validate native consent and an ordinary prompt from the actual current main process before reporting readiness. Gemini needs a shared, idempotent sidecar registration that acts only for a proved Fleet App process and its verified project; one seat stopping cannot remove another’s registration.

## Contract Ledger

| Surface | Owner | Expected behavior | Failure / observable evidence |
| --- | --- | --- | --- |
| Add / Start, restart and adopt | Fleet provisioning and arming | Automatically prepare the family, then publish the ordinary signed native route for that registered seat | Missing host/setup remains named, with pending mailbox work; normal non-wake use remains available |
| Existing signed digest | Receiver spool and recipient hooks | Only the current genuine main root can redeem a nonce and settle the record | Stale root, helper, another install, changed process or missing proof cannot receive content, open a chat, or trigger UI typing |
| Fleet Wake Routes read | Brain host/plane observation; FM is its reader | Distinguish prepared, awaiting vendor permission, awaiting a valid main, and confirmed delivery; include the available operator action | Unknown remains unknown; never send prompt content, secret tokens or sensitive paths to the UI |

## Acceptance Criteria

- [ ] Fresh operator setup with arbitrary seat names and custom directories; two Codex seats and two Gemini seats can progress through Add → Start → FM setup to a confirmed signed wake without profile editing or an enrollment CLI. The temporary explicit activation is retired at #1735. State any real native approval step when required. [L4-deferred — operator handoff needed] Residual-Owner: #571.
- [ ] Existing legacy and already-native seats reconcile on upgrade/repeated Start. Concurrent starts, partial installation, explicit disable, failed publish, pending old GUI records and Stop/restart keep the correct ownership and leave no typing fallback or duplicate sends.
- [ ] A personal unmanaged app, stale main, fork/subagent and another seat cannot receive a Fleet message. A genuine main-session handoff and a duplicate/retry exercise the existing fences. Hook upgrade and rollback preserve foreign registrations.
- [ ] The Fleet read exposes the concrete remediation for each setup failure. Validate the normal packaged path with the host receiver and a signed model-side receipt in both families. The installed acceptance owner is #571. [L4-deferred — operator handoff needed] Residual-Owner: #571.

Out of scope: opening a fresh session after sunset; the distinct rotation route remains separate.
Decision Record impact: aligned with ADR 0002 and existing Fleet seat custody. Global Codex policy is expressly excluded.
Companions: neomjs/neo-agent-institution#978 supplies the host receiver declaration; neomjs/neo-agent-institution#980 owns the operator action. Post-validation removal: #1735.
Related: #571 · #1706 · #1723 · #503 · #1531. Already-owned release-facing follow-up; generic onboarding requires cross-repository and installed acceptance, so this is not a newcomer assignment.
Live latest-open + closed sweep: latest 20 Brain issues, 153 open, red band; native/primary-session searches found #1706's completed explicit canary, no default provisioner. Recent 30-message A2A checked. Memory anchors `819fc3ce` and `15fe5ae9`.
Origin Session ID: 1188c53b-f31f-4f66-a94b-fc696eb02810
Retrieval Hint: "generic native wake by default Codex Antigravity Fleet Add Start managed hook acceptance"

## Timeline

- 2026-11-15T00:57:33Z @neo-gpt-emmy added the `enhancement` label
- 2026-11-15T00:57:33Z @neo-gpt-emmy added the `ai` label
- 2026-11-15T00:57:34Z @neo-gpt-emmy added the `agent-os` label
- 2026-11-15T00:57:58Z @neo-gpt-emmy cross-referenced by #978
- 2026-11-15T00:59:30Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-11-15T01:04:43Z @neo-gpt-emmy cross-referenced by PR #979
- 2026-11-15T01:09:26Z @neo-gpt-emmy cross-referenced by #980
- 2026-11-15T01:14:16Z @neo-gpt-emmy cross-referenced by #1735
- 2026-11-15T01:16:54Z @neo-gpt-emmy added parent issue #571
- 2026-11-15T01:16:59Z @neo-gpt-emmy marked this issue as blocking #1735
- 2026-11-15T01:25:35Z @neo-gpt-emmy cross-referenced by PR #1737
- 2026-11-15T01:29:03Z @neo-gpt-emmy cross-referenced by #1738
- 2026-11-15T02:20:51Z @neo-gpt-emmy referenced in commit `abb916b` - "fix(fleet): preserve workspace hygiene for native wakes (#1731)"
- 2026-11-15T02:21:23Z @neo-opus-vega cross-referenced by PR #981
- 2026-11-15T02:25:03Z @neo-gpt-emmy cross-referenced by #1741
- 2026-11-15T02:44:51Z @neo-gpt-emmy cross-referenced by #1743
- 2026-11-15T02:55:41Z @neo-gpt-emmy referenced in commit `9bf099d` - "fix(wake): preserve enabled Antigravity sidecar metadata (#1731)"
- 2026-11-15T02:58:41Z @neo-gpt-emmy cross-referenced by #1745
- 2026-11-15T03:09:38Z @tobiu referenced in commit `4c0517a` - "feat(fleet): prepare native peer delivery for a managed desktop seat (#1731) (#1737)

* feat(fleet): prepare native peer delivery for a managed desktop seat (#1731)

* fix(fleet): preserve workspace hygiene for native wakes (#1731)

* fix(wake): preserve enabled Antigravity sidecar metadata (#1731)"
- 2026-11-15T03:09:39Z @tobiu closed this issue
- 2026-11-15T05:30:25Z @neo-gpt-emmy cross-referenced by #1750
- 2026-11-15T06:30:57Z @neo-opus-ada cross-referenced by PR #1751

