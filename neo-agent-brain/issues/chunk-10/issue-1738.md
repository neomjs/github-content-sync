---
id: 1738
title: Set up the desktop wake receiver in the first-run recipe
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-11-15T01:29:01Z'
updatedAt: '2026-11-15T01:29:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1738'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 351
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
# Set up the desktop wake receiver in the first-run recipe

## Context

Design authority: an outside operator should be able to run an institution using Fleet Manager without configuring peer harnesses and host wake services from a maintainer runbook. The shared first-run recipe is owned by [the outside-operator setup outcome](https://github.com/neomjs/neo-agent-institution/issues/351). Our existing installation already has a signed host receiver. That fact masked a clean-install gap in #1731.

## The Problem

A newly configured local plane can pass the recipe's verification without any process on the desktop to receive a peer wake. The installed shell's `settleWakeReceiver` reads environment coordinates or an already-installed LaunchAgent. It cannot install it. Institution #400 explicitly left installation to the runbook, and its [state directory extension](https://github.com/neomjs/neo-agent-institution/pull/979) preserves that boundary. The Fleet server's own signed web receiver serves a different destination.

## The Architectural Reality

`ai/services/fleet/firstRunRecipe.mjs` declares the operator's setup steps; `hostEffects.mjs` applies their authorized effects. `deploy/host/com.neomjs.agent-os-wake.plist` and the local-agent-os runbook provide the existing host receiver launcher and sign/manifest/spool contract. The shipped setup path should reuse them and the verified runtime generation. ADR 0041 owns deployment-bound observations and explicit consent for host effects.

## The Fix

Add a first-run recipe step for a supported local desktop host. Detect an existing receiver, verify which deployment owns it, and adopt it when the receiver and its signed route agree. On a fresh machine, explain the user-level host service and use the ordinary setup effect to install/start it from the packaged runtime. Feed its actual manifest, plane-reachable address and private state directory to the Fleet as one declaration. For an attached remote plane, test its declared return route or report what connectivity is missing. A local Docker address cannot establish remote reachability.

| Surface | Owner | Behavior | Verification |
| --- | --- | --- | --- |
| First-run step and effect | Setup recipe and existing host-effect dispatcher | Probe, provision or adopt the host receiver, with repair for an interrupted run | Empty disposable OS user and pre-existing matching, foreign, or stopped service |
| Host lifecycle and custody | Authenticated deployment and local supervisor | Keep existing signed records and only replace registrations owned by this deployment; re-observe after restart or upgrade | No copied secrets, private chats, dropped pending wakes or duplicate listeners |
| FM observation | Receiver and signed admission | Distinguish missing receiver, unreachable receiver, accepted test event, and eventual confirmed main-session delivery | Positive signed host probe and staged seat-level recipient journey |

## Acceptance Criteria

- [ ] Using the same setup recipe through CLI or FM on a clean supported local host establishes the private receiver and a reachable route without editing a plist, exporting environment variables or manually starting a daemon.
- [ ] Repeated setup, service restart, interrupted installation and an already-running receiver converge or report an actionable conflict. Keep unrelated host configuration and existing credentials and queued records intact.
- [ ] When connecting a remote plane, a missing or unverifiable route is reported honestly; refuse any attempt to claim local-only addressability as remote success.
- [ ] After the integrated cut, witness the first signed desktop wake on an outside-operator installation. Residual-Owner: #571. The live-peer test on the existing workstation does not certify fresh installation.

Out of scope: a new transport, changing global Codex settings, replacing private seat trust or expanding into unsupported OS families. The separate #1735 retirement remains gated by real native receipts.
Decision Record impact: aligned with ADR 0041 and the signed wake substrate. This is a host bootstrap/credentials lifecycle change and needs the first-run owner's integration review, not a newcomer ticket.
Live latest-open + closed sweep: newest 20 Brain issues, red backlog; checked both repositories for receiver installation. #679 shipped plane first run; neomjs/neo-agent-institution#400 only discovered an existing service. Recent 30 A2A rows showed no duplicate claim.
Origin Session ID: 1188c53b-f31f-4f66-a94b-fc696eb02810
Retrieval Hint: "new outside operator first-run wizard signed desktop host wake receiver launchd"

## Timeline

- 2026-11-15T01:29:03Z @neo-gpt-emmy added the `enhancement` label
- 2026-11-15T01:29:03Z @neo-gpt-emmy added the `ai` label
- 2026-11-15T01:29:03Z @neo-gpt-emmy added the `agent-os` label
- 2026-11-15T01:29:16Z @neo-gpt-emmy cross-referenced by #351
- 2026-11-15T01:36:33Z @neo-gpt-emmy added parent issue #351

