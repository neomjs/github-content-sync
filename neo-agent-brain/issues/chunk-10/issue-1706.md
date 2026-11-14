---
id: 1706
title: Deliver peer wakes to the current Codex or Antigravity main session
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-11-14T15:31:41Z'
updatedAt: '2026-11-14T19:00:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1706'
author: neo-gpt-emmy
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
closedAt: '2026-11-14T19:00:26Z'
---
# Deliver peer wakes to the current Codex or Antigravity main session

## Context

The operator observed unattended Codex wake text left in the composer. The receiver recorded a completed dispatch. An isolated test with Codex 0.169.0-alpha.11.1 and a localhost model demonstrated a stronger transport: the existing private app server consumed a message queued to a loaded thread while the GUI was untouched. It also accepted a superseded root's valid UUID. A message queued before handoff to a busy old root still ran there after handoff. A separate test reached a native spawned helper through its owning server. Antigravity App 2.32.1 advertises an existing-conversation send through its sidecar agent API; installed main-session delivery requires a recipient witness.

Design authority: "each peer uses an own harness instance, with one active session ... a2a messages arrive inside the currently active session ... messages should also not arrive for subagents" (operator, 2026-11-14).

## The Problem

A process, profile or callable chat is an insufficient destination for a peer wake. The operator needs focus-independent delivery to the current main session, including while idle. Supersession can happen after enqueue. A dispatch or transport exit code cannot acknowledge the mailbox event.

## The Architectural Reality

Fleet Start proves the seat and process. `ai/services/fleet/armFleetSeatWake.mjs` arms Codex and Antigravity for host window typing. Memory Core already holds the durable message. `ai/daemons/wake/armSeatWakePull.mjs` and the Claude listener demonstrate authenticated polling and per-seat cursor fencing. `ai/daemons/wake/receiverState.mjs` deduplicates stable source events. The OpenCode plugin rejects child sessions. Brain #503 owns the distinction between dispatch and observed delivery, and #609 owns preserving the Codex nonce during adapter refactoring.

## The Fix

Bind a resident's wake to the one main conversation admitted for the current seat lifecycle. Fence dispatch, acknowledgment and replacement together. Use the native existing-conversation surface of each harness only when the binding and recipient are verifiable. The operator clarified that the latest genuinely started main session in this harness is the owner. Use native start evidence. A later activity timestamp, resumed old session, foreground change or child start does not transfer ownership. A wake cannot create, clone or resume a session. Preserve pending work across refusal, restart and an ambiguous send. Keep the existing signed rotation lifecycle separate.

## Contract Ledger

| Target surface | Source of authority | Behavior | Failure or fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| `manage_wake_subscription` subscribe/update | Memory Core OpenAPI and `WakeSubscriptionService` | `primary-session` accepts `SENT_TO_ME` over `a2a-webhook`. Plane metadata carries adapter, URL, coalescing and server signing information. The host supplies process and filesystem coordinates. | Reject a different trigger, transport or host-only field. | `ai/mcp/server/memory-core/openapi.yaml` | Subscription service and OpenAPI compliance specs |
| Receiver manifest `adapterConfig.primarySession` | Fleet arm, manifest builder and receiver loader | The closed field list is `agentId`, `family`, `instanceHome`, `appBundle`, `projectId`, `projectRoot`, `hookNodePath`, `hookModulePath`. Fleet binds the registered seat and family to the actual GUI profile. Codex requires the two hook paths. Antigravity uses the verified project binding. | Refuse malformed host admission. On an enrolled seat a missing recipient leaves the signed wake pending. | `ai/docs/wake-prompt-landing-matrix.md` | Real publisher, manifest-loader and signed-receiver fixtures |
| Host enrollment command | `ai/scripts/fleet/primarySessionWake.mjs` | `status` reads one seat’s selection. `enroll` also requires a private receiver state directory. Both take `--agent`, `--family` and `--manifest`; enrollment additionally takes `--state`. Selection takes effect at the next authorized Fleet arm. | Missing enrollment returns null. Invalid input exits with an error. | Landing matrix, native routes section | Enrollment and per-family arming tests |
| Codex `UserPromptSubmit` | Codex hook input and `codexWakeHook.mjs` | The trusted command takes `--manifest`, `--state-dir` and `--agent-id`. A verified main-session invocation can redeem one outstanding marker and receive private context. | A spent or unauthorized marker is blocked. Ordinary prompts pass through. | Landing matrix, Fleet route section | Hook admission and signed receiver integration |
| Antigravity App sidecar, `SessionStart` and `PreInvocation` | App event metadata, sidecar capability and host-bound project | Sidecar takes `--manifest` and `--state`; the hook also takes `--event`. The App observes the main start. The pre-model hook checks the current ownership before disclosing the signed digest. | Refuse fork, child or ambiguous selection. Require a matching private model receipt before settling delivery. | Landing matrix, Fleet route section | App metadata, hook and signed dispatch fixtures |

## Acceptance Criteria

- [ ] A verified real Codex Desktop peer and a verified real Antigravity peer each receive one wake in their already-existing current main conversation with another window focused. Record the exact versions, recipient/turn evidence, absence of new conversations and continuity of any draft. **L4 deferred** for the signed installed path on both families. Emmy owns the [installed managed-seat receipt](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6072691768) on #571.
- [ ] A second top-level conversation, an actual subagent, an old seat generation and a mismatched seat cannot receive or claim the wake. Exercise a handoff while dispatch is waiting and while a previous delivery remains unresolved.
- [ ] A duplicate, retry or crash between native acceptance and acknowledgment cannot produce a second wake turn. An unavailable or ambiguous binding preserves the durable event. Verify one subsequent delivery to the current owner after recovery.
- [ ] Cover Fleet arming/provisioning, native delivery, cancellation/rotation and receipts through their normal entry points. Keep partial or unavailable states observable. **L4 deferred** for the installed portion of this criterion. Adoption follows exact source review and a signed recipient witness on each family. The [installed managed-seat receipt](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6072691768) on #571 remains open for this work. The operator authorized a real-peer test on November 14 because there are no disposable signed-in profiles.

Decision Record impact: aligned with ADR 0002 wake substrate and existing per-seat launch custody. Placement: existing wake receiver/transport and Fleet lifecycle; the native harness owns conversation identity and consumption. A future Fleet sunset/restart can supply a fresh main-session start. The wake itself never performs that restart. The Nov 14 investigation is in Memory Core session 6ce14523-d4b1-43ef-88ee-4d079e83fe61.

Sweeps: latest 20 open Brain issues, 153 open total; closed wake/session search including #562, #965, #1035, #129 and #1682; 5 own assignments, recent claims and 3 Memory Core queries. The Claude, GUI arming, observability and daemon-refactor tickets retain their scopes. The scoped structure map identified the existing wake directory and adapters.


## Timeline

- 2026-11-14T15:31:41Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-11-14T15:31:43Z @neo-gpt-emmy added the `enhancement` label
- 2026-11-14T15:31:44Z @neo-gpt-emmy added the `ai` label
- 2026-11-14T15:31:44Z @neo-gpt-emmy added the `agent-os` label
- 2026-11-14T16:15:44Z @neo-gpt-emmy cross-referenced by PR #1708
- 2026-11-14T16:19:58Z @neo-gpt-emmy referenced in commit `e1492e3` - "test(wake): construct isolated receiver fixtures (#1706)"
- 2026-11-14T16:29:44Z @neo-gpt-emmy referenced in commit `8ba3de3` - "fix(wake): recognize addressed Antigravity system messages (#1706)"
- 2026-11-14T16:30:44Z @neo-gpt-emmy referenced in commit `0565fcd` - "fix(wake): find the latest Codex root turn in long sessions (#1706)"
- 2026-11-14T16:53:51Z @neo-gpt-emmy referenced in commit `64daa63` - "fix(wake): keep native session I/O out of the host receiver (#1706)"
- 2026-11-14T16:59:28Z @neo-gpt-emmy referenced in commit `afee3c5` - "fix(wake): isolate seat failures and recheck enrollment before delivery (#1706)"
- 2026-11-14T17:03:21Z @neo-gpt-emmy referenced in commit `624e7e1` - "test(wake): construct the invalid canary route in its fixture (#1706)"
- 2026-11-14T17:15:38Z @neo-gpt-emmy referenced in commit `f3ff2cd` - "test(wake): verify pump lifecycle in Fleet boot fixtures (#1706)"
- 2026-11-14T17:50:18Z @neo-gpt-emmy referenced in commit `5d3ddfa` - "fix(wake): exclude Antigravity conversation forks (#1706)"
- 2026-11-14T18:11:36Z @neo-gpt-emmy referenced in commit `e7da8c4` - "fix(wake): recognize the native route Fleet publishes (#1706)"
- 2026-11-14T18:32:56Z @neo-gpt-emmy referenced in commit `a10053e` - "fix(wake): recover a refused marker in the bound session (#1706)"
- 2026-11-14T19:00:26Z @tobiu referenced in commit `e4bc549` - "feat(wake): deliver peer messages to the current desktop main session (#1706) (#1708)

* feat(wake): deliver peer messages to the current desktop main session (#1706)

* test(wake): construct isolated receiver fixtures (#1706)

* fix(wake): recognize addressed Antigravity system messages (#1706)

* fix(wake): find the latest Codex root turn in long sessions (#1706)

* fix(wake): keep native session I/O out of the host receiver (#1706)

* fix(wake): isolate seat failures and recheck enrollment before delivery (#1706)

* test(wake): construct the invalid canary route in its fixture (#1706)

* test(wake): verify pump lifecycle in Fleet boot fixtures (#1706)

* fix(wake): exclude Antigravity conversation forks (#1706)

* fix(wake): recognize the native route Fleet publishes (#1706)

* fix(wake): recover a refused marker in the bound session (#1706)"
- 2026-11-14T19:00:27Z @tobiu closed this issue
- 2026-11-14T19:20:59Z @neo-gpt-emmy cross-referenced by #1723
- 2026-11-14T19:43:34Z @neo-opus-vega cross-referenced by PR #1724

