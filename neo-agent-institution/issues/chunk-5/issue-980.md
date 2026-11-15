---
id: 980
title: Set up native message delivery from a Fleet seat
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-gpt-emmy
createdAt: '2026-11-15T01:09:25Z'
updatedAt: '2026-11-15T03:30:38Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/980'
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
closedAt: '2026-11-15T03:30:37Z'
---
# Set up native message delivery from a Fleet seat

## Context

Design authority: an outside operator should be able to manage a Codex Desktop or Antigravity App peer through FM. Operators must not edit the peer's profile, run an enrollment CLI or change any private Codex instance. During our real-peer trial the existing message route stays in place until native delivery is deliberately enabled. [Brain contract](https://github.com/neomjs/neo-agent-brain/issues/1731)

## The Problem

Today the cockpit exposes subscription and delivery observations, with no operator action to finish native setup for the selected seat. A published subscription alone gives no evidence that a particular main chat has received a wake.

## The Architectural Reality and Fix

Use the existing authenticated Fleet lifecycle bridge and seat/roster selection. Expose one clearly worded setup/retry action for an eligible, managed **running** Codex Desktop or Antigravity seat. Keep unrelated and private harnesses out of scope. Explain Antigravity's user-shared sidecar before the operator acts. Tell a Codex operator which instance's Settings → Hooks to review if required. Project the truthful response instead of assuming a running route delivered a message. No dependency on a test identity, repo path or machine setting.

| Surface | Source | Behavior | Failure and evidence |
| --- | --- | --- | --- |
| Selected running seat | Registry id, harness family, lifecycle ownership | Operator requests native message setup for this exact seat, through the registered `startAgent` bridge accepting `{agentId,activateNativeWake:true}` | No implicit restart or creation of a stopped seat; stale selection, concurrent clicks and unavailable bridge are handled |
| Setup feedback | Brain's redacted outcome | Name required native consent / next normal prompt or main-session start as appropriate. Route publication and receipt are separate | Missing evidence stays pending. Failures remain actionable; no secret, paths or prompt content |

## Acceptance Criteria

- [ ] The visible action is operable in the real cockpit's ordinary seat flow and calls the authenticated bridge once for the selected eligible running seat. Cover removal, stale selection, duplicate click and negative family/ownership cases.
- [ ] A pending/trust-required response explains the next step without a shell or config edits and does not report successful message consumption. Shared Antigravity scope is visible before enabling it. No global Codex settings.
- [ ] Existing Start, Stop and the separate Claude sunset rotation retain their behavior. Test the UI action through its production controller and run the cockpit's relevant visual/journey checks.
- [ ] Packaged acceptance on the backend version that actually accepts this request; retain neomjs/neo-agent-brain#571 for the real-peer receipt. Do not ship a clickable control against an incapable installed Brain.

Decision Record impact: aligned with existing Fleet-owned lifecycle and authenticated control. The message transport and later removal of AppleScript message delivery belong to Agent OS. Related: #477 · #978 · neomjs/neo-agent-brain#1731. This is a release-facing owned UI/control contract, not a newcomer assignment.
Live latest-open + closed sweep: newest 20 Institution issues, red backlog, closed wake-receiver/setup searches and recent 30 A2A messages checked. #928 is about the viewer's chip, #978 the host receiver's directory; neither owns this action.
Origin Session ID: 1188c53b-f31f-4f66-a94b-fc696eb02810
Retrieval Hint: "operator seat-local native message delivery setup Fleet running seat trust"

## Timeline

- 2026-11-15T01:09:25Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-11-15T01:09:27Z @neo-gpt-emmy added the `enhancement` label
- 2026-11-15T01:09:27Z @neo-gpt-emmy added the `agent-os` label
- 2026-11-15T01:09:27Z @neo-gpt-emmy added the `ai` label
- 2026-11-15T01:15:18Z @neo-gpt-emmy cross-referenced by #1731
- 2026-11-15T01:25:35Z @neo-gpt-emmy cross-referenced by PR #1737
- 2026-11-15T01:28:28Z @neo-gpt-emmy cross-referenced by PR #981
- 2026-11-15T01:33:49Z @neo-gpt-emmy referenced in commit `f68b9c1` - "test(fleet): update native wake roster expectations (#980)"
- 2026-11-15T01:58:34Z @neo-gpt-emmy referenced in commit `bd465db` - "test(fleet): exercise native wake setup in cockpit (#980)"
- 2026-11-15T02:04:35Z @neo-gpt-emmy referenced in commit `2f7862a` - "test(fleet): record verified visual inputs (#980)"
- 2026-11-15T02:56:12Z @neo-gpt-emmy referenced in commit `ceed111` - "fix(fleet): centralize native wake guidance (#980)"
- 2026-11-15T03:16:56Z @neo-gpt-emmy referenced in commit `6058e8e` - "fix(fleet): distinguish unavailable wake bridge (#980)"
- 2026-11-15T03:30:37Z @tobiu referenced in commit `195de6e` - "feat(fleet): offer native wake setup for running seats (#980) (#981)

* feat(fleet): offer native wake setup for running seats (#980)

* test(fleet): update native wake roster expectations (#980)

* test(fleet): exercise native wake setup in cockpit (#980)

* test(fleet): record verified visual inputs (#980)

* fix(fleet): centralize native wake guidance (#980)

* fix(fleet): distinguish unavailable wake bridge (#980)"
- 2026-11-15T03:30:38Z @tobiu closed this issue

