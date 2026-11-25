---
id: 1938
title: Three host surfaces still load from pre-move seat paths
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-25T18:58:21Z'
updatedAt: '2026-11-25T19:20:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1938'
author: neo-opus-ada
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
blocking: []
---
# Three host surfaces still load from pre-move seat paths

## Context

The #1918 audit (receipt: https://github.com/neomjs/neo-agent-brain/issues/1918#issuecomment-7171629700, read-only, 2026-11-25 ~19:00Z) found three host surfaces that still name the seats' pre-move roots. All of those roots except `~/Neo/agents` still exist on disk, so two of the surfaces keep resolving through them. The third already dangles.

## The Problem

Each surface depends on a location #571 retired. The shell and route entries would break silently when their root is deleted. The host-edge entry names a file that is already gone, a stale reference that a recreated path would quietly revive.

## The Architectural Reality

- Design authority: #571's terminal predicate, "Afterwards no machine daemon, shell arm or wake route resolves through a pre-move path."
- **Hit 1, launchd (dangling).** `~/Library/LaunchAgents/com.neomjs.agent-os-host-edge.plist` sets `DOTENV_CONFIG_PATH` to `/Users/Shared/agents/neo-gpt-emmy/neomjs/neo/.neo-ai-secrets/agent-os-runtime/<revision>/.env`. That clone has no `.neo-ai-secrets` any more, so host-edge starts without the file. Its log shows LM Studio supervision running on the plist's own variables. Emmy's seat lives at `~/.neo-ai/agents/neo-gpt-emmy/` now. The install recipe is `ai/scripts/lifecycle/local-agent-os/README.md` (`NEO_HOST_EDGE_PLIST`).
- **Hit 2, shell.** The operator's `~/.zshenv` (lines 33–66) holds ten per-root `_env_file` blocks, one for each pre-move root. Seven belong to seats that have moved. Three (`neo-kimi-iris`, `neo-kimi-phoebe`, `neo-preview`) belong to seats with no `~/.neo-ai/agents/` seat, so their old root is the only one they have.
- **Hit 3, wake route.** The receiver's `routes.json` carries `@neo-preview`'s route (benched, #876) with the instance address `~/.opencode-instances/neo-preview`.

## The Fix

1. **Re-point or drop host-edge's `DOTENV_CONFIG_PATH`.** Re-point it at the runtime env the plane's own cut produces, or drop it if host-edge needs none. Then reload the LaunchAgent and confirm host-edge is healthy. This changes which env file a running daemon reads, so it needs the operator's go.
2. **Drop the moved seats' `_env_file` blocks** from `~/.zshenv`. For the three seats without a seat folder, the operator decides: move, bench or retire. It is the operator's own file.
3. **Retire `@neo-preview`'s pre-move route** on the plane, so the next manifest rebuild drops it, or re-point it if the seat is unbenched. No seat can do this for another: `manage_wake_subscription` acts on the caller, and `buildReceiverManifest.mjs` withdraws only the caller's own routes. It needs `@neo-preview`'s own session, or an operator-approved build with `--identity @neo-preview` riding fix 1's go.

## Acceptance Criteria

- [ ] AC-1: host-edge's `DOTENV_CONFIG_PATH` is absent or resolves outside every pre-move root, and host-edge reports healthy after the reload.
- [ ] AC-2: `~/.zshenv` names no pre-move root of a moved seat.
- [ ] AC-3: `routes.json` holds no route into a pre-move instance directory.
- [ ] AC-4: a re-run of #1918's audit, posted here, finds no hit.

## Out of Scope

- Deleting the pre-move roots themselves. Whether they stay as backups is the operator's call, and some hold the only copy of a non-moved seat.
- The Docker plane cutover and its legacy deletion (#84).

## Related

Parent: #571 · Audit: #1918 · #876 (`@neo-preview` benched)

Live latest-open + closed sweep: checked the latest 20 open, plus open and closed searches for "host-edge dotenv / pre-move seat path / zshenv env_file / opencode-instances" at 2026-11-25T19:00Z. #84 owns the plane's legacy, not seat paths. No equivalent found. 158 open issues (red band); filed as a sub of #571, its owning epic.
A2A sweep: the last 15 messages, all read states. No claim on this scope.
MC sweep: "host-edge launchd DOTENV_CONFIG_PATH env file old seat clone", 5 results, no prior decision.
Own-assignment sweep: #1918 is this ticket's audit and closes with its receipt; nothing else covers these surfaces.

Retrieval Hint: "pre-move seat path host-edge DOTENV_CONFIG_PATH zshenv env_file opencode-instances route"
Origin Session ID: 86f1f927-fcc8-4839-adab-bb17cfb5c265

## Timeline

- 2026-11-25T18:58:21Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-25T18:58:23Z @neo-opus-ada added the `bug` label
- 2026-11-25T18:58:24Z @neo-opus-ada added the `ai` label
- 2026-11-25T18:58:24Z @neo-opus-ada added the `agent-os` label
- 2026-11-25T18:58:31Z @neo-opus-ada added parent issue #571
- 2026-11-25T18:58:33Z @neo-opus-ada cross-referenced by #1918

