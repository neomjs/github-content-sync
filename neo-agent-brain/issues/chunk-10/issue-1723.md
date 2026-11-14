---
id: 1723
title: Accept the loopback endpoint supplied by the installed Antigravity sidecar
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-11-14T19:20:58Z'
updatedAt: '2026-11-14T19:20:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1723'
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
---
# Accept the loopback endpoint supplied by the installed Antigravity sidecar

## Context

The first installed trial of #1708 observed Antigravity App 2.32.1 hot-load the native sidecar, then stop it before it could record a root census. A short-lived diagnostic on the App-launched sidecar in two running instances confirmed the selected App PID and the bundle-owned agent API executable. The App supplied its language-server endpoint as a bare loopback host and port. No address, credential, conversation content or model input was logged. The diagnostic was removed and the reviewed definition restored.

## Problem and owner

`validateAntigravitySidecarCapability` in `ai/daemons/wake/antigravityWakeSidecar.mjs` parses the address as an HTTP(S) URL. The actual bare endpoint fails that check. Separately, the App ancestor matcher can mistake the first part of an `Antigravity Helper` executable name for the main App and stop its search early. The route can be published while the recipient remains unknown.

Design authority is the operator's current-main-session requirement on #1706. The sidecar may dispatch only after it proves both its vendor capability and its registered Fleet App instance.

## Fix and acceptance

- [ ] Accept the observed bare loopback address with an explicit valid port and the previously accepted HTTP(S) loopback form. Refuse remote or wildcard addresses, user info, malformed ports, and a mismatched executable. Keep the vendor environment unchanged for dispatch.
- [ ] Walk past a vendor helper to the actual main executable. Keep the profile and process epoch checks, and refuse a missing or ambiguous main ancestor.
- [ ] Exercise the observed address shape, App and helper process chains, and negative controls with no vendor secret or actual model call. Emit a bounded startup diagnostic for distinguishing a failed capability or ancestry check from failure during dispatch.

The installed private recipient and draft/focus receipt remains on #571. Source repair can be checked with a disposable fixture and the sanitized observations above. There is no request here to restart a peer or create a conversation.

Related: #1706, #1708, #571. Decision record impact: aligned with ADR 0002. Scoped structure map: existing sidecar and its family-specific spec in `ai/daemons/wake` and `test/playwright/unit/ai/daemons/wake`. Live latest 20 open of 157 and closed Antigravity/wake sweep on 14 Nov, five own assignments and recent A2A claims, KB and Memory Core checked. The earlier work owns transport and installed acceptance; this leaf owns its newly observed bootstrap defect. Origin Session ID: d2972912-1050-4e40-8f00-ca193b527781

## Timeline

- 2026-11-14T19:20:58Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-11-14T19:21:00Z @neo-gpt-emmy added the `bug` label
- 2026-11-14T19:21:00Z @neo-gpt-emmy added the `ai` label
- 2026-11-14T19:21:00Z @neo-gpt-emmy added the `agent-os` label
- 2026-11-14T19:23:33Z @neo-gpt-emmy cross-referenced by PR #1724
- 2026-11-14T19:24:30Z @neo-gpt-emmy cross-referenced by #571

