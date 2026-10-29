---
id: 1340
title: The Fleet compose render test omits the wake-records variable it needs
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-28T22:53:45Z'
updatedAt: '2026-10-28T23:18:05Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1340'
author: neo-opus-ada
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
closedAt: '2026-10-28T23:18:05Z'
---
# The Fleet compose render test omits the wake-records variable it needs

## Context

`test/playwright/unit/ai/deploy/FleetServerComposition.spec.mjs` › "local MCP receivers admit only the Compose Host names their callers use" renders `deploy/cloud/docker-compose.yml` + `docker-compose.local-agent-os.yml` through `docker compose … config --format json` with a minimal env (two token files, a project name).

Since #530 (36cc0d73, 2026-09-26) the local overlay's `mc-server` bind source is `${NEO_WAKE_RECEIVER_RECORDS_HOST_DIR:?…}`. That is fail-closed on purpose, and `WakeReceiverRecordsMount.spec.mjs` pins it. The test's env never got the variable, so the render stops at interpolation and the host-allowlist assertions (`kb-server` → `kb-server`, `mc-server` → `mc-server,ingress`, never `*`) have not run since.

- CI: Brain Unit run 40679982173 on dev `05e42855` lists it among the 9 retained-suite reds.
- Local, dev `05e42855`, Docker Compose 5.1.4: the spec runs 1 failed, 6 passed, with `required variable NEO_WAKE_RECEIVER_RECORDS_HOST_DIR is missing a value`.

## The Problem

The test guards that each local MCP receiver admits only the Host names its callers use. A widening to `*` would currently go unseen.

## The Architectural Reality

- The overlay's comment states the contract: the env file supplies the variable, a missing variable fails the render, and a missing directory fails `up` (`create_host_path: false`).
- `config` only interpolates; it neither creates nor checks the host path. So any value satisfies the render, which is how the same env block already treats the token files (`/dev/null`).

## The Fix

Add `NEO_WAKE_RECEIVER_RECORDS_HOST_DIR` to that render's env, beside the token-file fixtures. The overlay and its `:?` stay as they are.

## Acceptance Criteria

- [ ] The render's env supplies `NEO_WAKE_RECEIVER_RECORDS_HOST_DIR`, and the compose files are unchanged.
- [ ] The test reaches its host-allowlist assertions and passes with Docker Compose available.
- [ ] On the PR, Brain Unit's base/head comparison drops this red and introduces none.

## Out of Scope

Rendering from the runbook's env-file template instead of an inline env; the other retained-suite reds (#1338 and the rest on #201).

## Related

Related: #201 · #530 (made the variable required)

Live latest-open + closed sweep: latest 20 open Brain issues + closed search ("FleetServerComposition wake receiver records compose render") + org search for the variable at 2026-10-28T22:53Z; only #530 (its origin) matches, no equivalent. A2A: last 10 messages, no claim on this scope. Own assignments: #201 is the umbrella; #1338 is a different spec.
Structure map: N/A (no file added or moved).
Decision Record impact: none.

Ownership: self-assigned, as a slice of #201.

Origin Session ID: 192574ae-badb-4712-8cd2-9e80f5566d22
Retrieval Hint: "FleetServerComposition NEO_WAKE_RECEIVER_RECORDS_HOST_DIR required variable compose render"

## Timeline

- 2026-10-28T22:53:45Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-28T22:53:46Z @neo-opus-ada added the `bug` label
- 2026-10-28T22:53:47Z @neo-opus-ada added the `ai` label
- 2026-10-28T22:53:47Z @neo-opus-ada added the `testing` label
- 2026-10-28T22:54:45Z @neo-opus-ada cross-referenced by PR #1341
- 2026-10-28T22:57:13Z @neo-opus-ada cross-referenced by #201
- 2026-10-28T23:18:05Z @tobiu referenced in commit `8c718b2` - "fix(test): the Fleet compose render supplies the wake-records variable (#1340) (#1341)

The local overlay's mc-server bind requires NEO_WAKE_RECEIVER_RECORDS_HOST_DIR (fail-closed since #530), and the host-allowlist render's minimal env never named it, so docker compose config stopped at interpolation before the kb/mc Host-name assertions ran. The env now supplies it beside the token-file fixtures; config only interpolates, so the value is not read."
- 2026-10-28T23:18:05Z @tobiu closed this issue

