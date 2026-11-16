---
id: 997
title: Fleet picks up the desktop wake receiver after first-run setup
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
assignees: []
createdAt: '2026-11-16T01:36:07Z'
updatedAt: '2026-11-16T01:36:07Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/997'
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
blockedBy:
  - '[ ] 1738 Set up the desktop wake receiver in the first-run recipe'
blocking: []
---
# Fleet picks up the desktop wake receiver after first-run setup

## Context

Design authority: an outside operator can provision a local plane or connect to a remote one through Fleet Manager. [Brain #1738](https://github.com/neomjs/neo-agent-brain/issues/1738) owns the shared first-run recipe and host receiver lifecycle. This leaf owns the installed shell that calls it. [First-run outcome](https://github.com/neomjs/neo-agent-institution/issues/351) · [Clean-host reproduction](https://github.com/neomjs/neo-agent-brain/issues/1738#issuecomment-6904271033) · [Maryam’s intake](https://github.com/neomjs/neo-agent-brain/issues/1738#issuecomment-6904769077)

## The Problem

Fleet can start before the receiver exists. Its child spawn and respawn use coordinates settled at the start of that boot. A successful Connect can also update a bearer in place. Either path can leave the operator with a usable plane and no evidence that a wake can get back to the desktop. [Discovery](https://github.com/neomjs/neo-agent-institution/blob/242498ddcf861ea45ca7fbe9c24d6e8aa6642444/harness/main.mjs#L1199-L1208) · [Spawn](https://github.com/neomjs/neo-agent-institution/blob/242498ddcf861ea45ca7fbe9c24d6e8aa6642444/harness/main.mjs#L1265-L1284) · [Connect](https://github.com/neomjs/neo-agent-institution/blob/242498ddcf861ea45ca7fbe9c24d6e8aa6642444/harness/planeConfig.mjs#L578-L615)

## The Architectural Reality

The packer stages Brain’s `ai` and `src`. The shared recipe already expects `deploy/cloud` and reads both Compose files before its first write. The launcher already resolves a verified runtime root and executable. A Finder-launched app should use these packaged inputs without a developer clone, an exported shell environment or a system Node. [Pack allowlist](https://github.com/neomjs/neo-agent-institution/blob/242498ddcf861ea45ca7fbe9c24d6e8aa6642444/harness/pack.mjs#L45-L49) · [Launcher](https://github.com/neomjs/neo-agent-institution/blob/242498ddcf861ea45ca7fbe9c24d6e8aa6642444/harness/brain.mjs#L93-L106) · [Layout](https://github.com/neomjs/neo-agent-brain/blob/1f884fed59406b481e22346ee9ca68731a59c354/ai/scripts/setup/firstRun.mjs#L123-L132) · [Pre-write read](https://github.com/neomjs/neo-agent-brain/blob/1f884fed59406b481e22346ee9ca68731a59c354/ai/services/fleet/setupOrchestration.mjs#L136-L140)

## The Fix and Contract Ledger

| Target surface | Authority | Behavior | Fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| Packaged Create | Verified generation and shared host layout | Ship the setup assets the recipe consumes, including the Compose inputs and necessary local dependencies. Supply the executable, runtime root and environment the receiver needs after login. | Report missing inputs before writing. Retain a generation still in use until the service has switched. | Installed setup guide | Isolated package with no checkout or system Node. Use the actual emitted plist in [Fleet’s reader](https://github.com/neomjs/neo-agent-institution/blob/242498ddcf861ea45ca7fbe9c24d6e8aa6642444/harness/wakeReceiver.mjs#L163-L238). |
| Live Fleet after setup | Fresh receiver declaration for the selected deployment | Re-read and deliver manifest, plane-reachable origin and private state directory together through a supported refresh. | Missing, malformed or conflicting evidence yields a reason and a next step. Preserve running seats and pending wakes. | Create / Recheck | Fleet starts with empty discovery. After Create the running consumer sees the new tuple. Repeat and upgrade are safe. |
| Remote Connect | Credential admission and a separate check from the selected plane | Keep a valid connection usable and report return-route evidence on existing status surfaces. | An undeclared or unreachable route leaves wakes unavailable with guidance. Local HTTP or a Docker-only name cannot prove remote reachability. | Connect guidance | Valid PAT with a working route, a missing route and an unreachable route, including in-place bearer rotation. |

## Acceptance Criteria

- [ ] Run Create from the staged package through the real shared layout and receiver entry point. The emitted declaration survives discovery and a generation change.
- [ ] A Fleet running before setup can use the new coordinates. Repeating Recheck starts zero additional services and preserves an existing pending wake.
- [ ] The remote Connect tests distinguish usable plane, verified return route, signed event acceptance and confirmed seat delivery. A missing wake route keeps the authenticated connection usable.
- [ ] Keep the outside-operator installation witness on [#534](https://github.com/neomjs/neo-agent-institution/issues/534) and installed seat acceptance on [Brain #571](https://github.com/neomjs/neo-agent-brain/issues/571).

**Out of scope.** A second installer, a new transport or tunnel, and a new cockpit dashboard. [#400](https://github.com/neomjs/neo-agent-institution/issues/400) and [#978](https://github.com/neomjs/neo-agent-institution/issues/978) shipped existing-service discovery and coordinates. [#727](https://github.com/neomjs/neo-agent-institution/issues/727) covers forge registration. [#980](https://github.com/neomjs/neo-agent-institution/issues/980) owns the seat action.

Decision Record impact: aligned with [ADR 0041](https://github.com/neomjs/neo-agent-brain/blob/1f884fed59406b481e22346ee9ca68731a59c354/learn/agentos/decisions/0041-bootstrap-record-verified-plane-handoff.md). This package/runtime integration needs maintainer review.

Live latest-open + closed sweep: latest open Institution queue, open and closed receiver/packaged-setup matches, 40 children of #351, recent A2A and memory/KB prior art. #534 explicitly excludes feature work. Emmy’s open Institution assignment is unrelated. Ownership belongs in the existing packer, setup broker, and Fleet boot/Connect paths.

Origin Session ID: 715cc9e5-d5e9-4715-9621-870fd2f3becd
Retrieval Hint: “Fleet starts before first-run installs a desktop wake receiver, remote Connect succeeds without a return route”

## Timeline

- 2026-11-16T01:36:08Z @neo-gpt-emmy added the `enhancement` label
- 2026-11-16T01:36:09Z @neo-gpt-emmy added the `agent-os` label
- 2026-11-16T01:36:09Z @neo-gpt-emmy added the `ai` label
- 2026-11-16T01:37:34Z @neo-gpt-emmy cross-referenced by #1738
- 2026-11-16T01:38:08Z @neo-gpt-emmy marked this issue as being blocked by #1738

