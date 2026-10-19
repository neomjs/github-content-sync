---
id: 1170
title: A plane-attached Codex Desktop Start refuses its own Neural Link launcher row at the prepared-adapter inspection
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-19T02:57:22Z'
updatedAt: '2026-10-19T03:24:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1170'
author: neo-opus-vega
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
closedAt: '2026-10-19T03:24:57Z'
---
# A plane-attached Codex Desktop Start refuses its own Neural Link launcher row at the prepared-adapter inspection

## Context

With the #1167 project check out of the way (tonight via Mnemosyne's workaround), `neo-gpt-euclid`'s Start on the new FM (Brain e4886d2a) fails one step later: `dispatch of 'startAgent' failed: FleetLifecycleService.inspectPreparedRemoteMcpAdapter: installed 'codex-desktop' did not preserve 'neo-mjs-neural-link' local stdio execution contract` (2026-10-19 02:54:22Z, `FleetLifecycleService.mjs:3539`, from `startAgentProvisioned.mjs:730`). This hits every plane-attached Codex Desktop seat (`mcpTarget.kind === 'tenant'`): Euclid, Sophie, Emmy.

## The Problem

`inspectPreparedRemoteMcpAdapter` runs the installed Codex against the prepared `CODEX_HOME` and requires every local stdio row's `transport.env` to deep-equal `NODE_RUNTIME_ENV` (`{ELECTRON_RUN_AS_NODE: '1'}`). Since #1156, a Codex Desktop seat's `neo-mjs-neural-link` row is Fleet's launcher table in the seat's home config. Its env carries `NEO_AGENT_IDENTITY`, `NEO_FLEET_LAUNCH_ISSUER`, `NEO_FLEET_LAUNCH_GRANT` and the resident MCP paths (19 keys on Euclid's real file). #1156 did not teach the inspection about launched rows, and its arms run Codex Desktop without a tenant target, so the gap stayed green.

Design authority: #1145's design (a Codex Desktop seat starts Neural Link through the issuer; its row carries the identity, placement, issuer and a per-Start grant) and the inspection's own stated purpose (the installed adapter parsed Fleet's projection, maps only MC and KB to the plane, and forwards nothing from the host).

## The Fix

For a server the seat starts through the issuer, compare the installed row with the launcher table Fleet rendered for this Start (command, args, and env key set, with the grant compared by presence, not value) instead of `NODE_RUNTIME_ENV`. The other stdio rows keep today's contract.

## Acceptance Criteria

- [ ] A tenant-target Codex Desktop Start whose Neural Link row is Fleet's launcher table passes the inspection.
- [ ] A launched row whose command, args or env key set differs from Fleet's rendering still refuses.
- [ ] Non-launched stdio rows keep the `NODE_RUNTIME_ENV` contract.
- [ ] [L3-deferred — operator handoff needed] Installed: Euclid starts on the next FM candidate. Residual-Owner: neomjs/neo-agent-institution#736.

Related: #1145 / #1156, #1167, #1169, neomjs/neo-agent-institution#732.



## Timeline

- 2026-10-19T02:57:23Z @neo-opus-vega added the `bug` label
- 2026-10-19T02:57:24Z @neo-opus-vega added the `ai` label
- 2026-10-19T03:02:57Z @neo-fable-mnemosyne assigned to @neo-fable-mnemosyne
- 2026-10-19T03:07:08Z @neo-opus-vega cross-referenced by PR #1171
- 2026-10-19T03:07:19Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-19T03:08:43Z @neo-fable-mnemosyne unassigned from @neo-fable-mnemosyne
- 2026-10-19T03:11:22Z @neo-fable-mnemosyne cross-referenced by PR #1168
- 2026-10-19T03:15:19Z @neo-opus-vega referenced in commit `f2ae363` - "test(fleet): a plane-attached Codex Desktop Start hands the inspection exactly the servers its binding launches (#1170)"
- 2026-10-19T03:24:57Z @tobiu referenced in commit `6b01f3a` - "fix(fleet): a plane-attached Codex Desktop Start reads its launched Neural Link row as Fleet's launcher table (#1170) (#1171)

* fix(fleet): a plane-attached Codex Desktop Start reads its launched Neural Link row as Fleet's launcher table (#1170)

The prepared-adapter inspection held every local stdio row to the plain Node
runtime env, but a Codex Desktop seat's Neural Link row has been Fleet's launcher
table since the launch-admission move. Start now passes the servers it launches
through the issuer, and the inspection checks those rows against the launcher
contract the renderer writes. LAUNCHER_ENTRYPOINT moves to the plan module so
both read one value.

* test(fleet): a plane-attached Codex Desktop Start hands the inspection exactly the servers its binding launches (#1170)"
- 2026-10-19T03:24:57Z @tobiu closed this issue
- 2026-10-19T03:29:53Z @neo-fable-mnemosyne cross-referenced by #1173

