---
id: 373
title: The orchestrator container never passes its healthcheck or reaps git
state: CLOSED
labels:
  - bug
  - ai
  - build
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-09-19T11:52:24Z'
updatedAt: '2026-09-19T12:17:24Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/373'
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
closedAt: '2026-09-19T12:17:24Z'
---
# The orchestrator container never passes its healthcheck or reaps git

## Context

Measured 2026-09-19 in #253's isolated rehearsal: disposable project `neo-brain-proof-253`, no published ports, project-scoped volumes, all three sync producers off. Four images were built from `deploy/cloud` at Brain `d5ae3e8767`. KB, MC, Fleet and Chroma went healthy within 16 s, and every running container reads `d5ae3e876786405902f0dc9c47d532504e115b42` from `/app/.neo-revision`. **The orchestrator stayed `unhealthy`.** This is the amber @neo-opus-vega recorded in #12's rehearsal at `90d41ff` (2026-08-30) and could not isolate. #253's acceptance requires "KB, MC, Fleet, and Orchestrator health/probe checks pass on the Brain cohort", so the cut would fail acceptance deterministically and roll back.

## The Problem

**1. The healthcheck cannot run in a Brain image.** `deploy/cloud/docker-compose.yml` (orchestrator `healthcheck.test`) starts with `await import('./src/Neo.mjs');await import('./src/core/_export.mjs')`. Those are Engine-checkout paths. A Brain image has the Engine at `node_modules/neo.mjs/`, and the Brain repo has no `src/Neo.mjs` at all. Every probe throws before its `try`:

```text
Error [ERR_MODULE_NOT_FOUND]: Cannot find module '/app/src/Neo.mjs' imported from /app/[eval1]
```

The orchestrator itself is fine. The same probe with package specifiers (`neo.mjs/src/Neo.mjs`, `neo.mjs/src/core/_export.mjs`), run in the same container, returns `{"fresh":true}` from `inspectAuthorityLease`.

**2. PID 1 never reaps the orchestrator's `git` children.** The rehearsal orchestrator had 1732 PIDs within five minutes of boot, 1721 of them zombie `git` processes re-parented to PID 1 (`MainThread`, node), with a stable count afterwards. The live pre-split orchestrator (Engine `467fd122f3`, one day up) shows the same class at 63 zombie `git`. Node as PID 1 does not reap orphaned grandchildren, and the service declares no `init`. Nothing sets `pids_limit` today, so this is a leak and not yet an outage.

**Why CI never saw it:** `test/playwright/unit/ai/daemons/orchestrator/daemon.spec.mjs` pins the probe by *text* (`toContain("await import('./ai/config.mjs')")`, `toContain('inspectAuthorityLease')`) and never resolves what it imports. A string assertion over an executable command stays green while the command cannot run.

## The Architectural Reality

- The probe is the orchestrator's liveness contract: `inspectAuthorityLease({dir: AiConfig.orchestrator.dataDir, profile: AiConfig.orchestrator.authorityProfile})` → exit 0 while the per-role authority lease is fresh.
- The Engine reaches Brain code as the `neo.mjs` package, which is how the daemon spec itself imports it (`import 'neo.mjs/src/Neo.mjs'`).
- The other services' probes are repo files (`ai/scripts/diagnostics/mcpHealthcheck.mjs`, `fleetHealthcheck.mjs`), which is why they survived the split. The orchestrator's is inline `-e` source, which no import resolution ever checked.
- Of the four services, only the orchestrator spawns `git`: KB, MC and Fleet ran at 11 PIDs each in the same rehearsal.

## The Fix

1. In `deploy/cloud/docker-compose.yml`, the orchestrator probe imports `neo.mjs/src/Neo.mjs` and `neo.mjs/src/core/_export.mjs` (package specifiers, resolved from `/app/node_modules`).
2. The orchestrator service declares `init: true`, so Docker's init reaps re-parented children.
3. `daemon.spec.mjs` gains an arm that **resolves** every specifier the probe imports from the package root, and asserts the service declares `init: true`. It must be red on `dev`, where `./src/Neo.mjs` does not exist.

## Acceptance Criteria

- [ ] The orchestrator probe imports resolve inside a Brain image. Evidence: a rehearsal (isolated as #253 prescribes) reports the orchestrator `healthy`.
- [ ] The orchestrator declares `init: true`, and the rehearsal's zombie `git` count stays at 0 after boot.
- [ ] A unit arm resolves every module the probe imports and fails on `dev`'s probe. The string assertions stay, or are replaced by it.
- [ ] No other service's probe or `init` changes.

## Out of Scope

- Why the orchestrator runs roughly 1700 `git` commands in its first minute with all three sync producers off. That is a separate question, recorded here as an observation. With reaping fixed it stops leaking PIDs, and its cost is its own ticket if it matters.
- The live pre-split cohort (#253 replaces it).
- #237's tenant `REF_NOT_FOUND` retries.

## Avoided Traps

- ⛔ **Relaxing the healthcheck to make it green** (a longer `start_period`, or treating it as optional). The probe is right about what it checks; only its imports are wrong.
- ⛔ **Another string assertion.** That shape is what kept this red in every Brain image since the split while CI stayed green. The arm has to resolve the imports.
- ⛔ **`init: true` on every service by reflex.** Only the orchestrator spawns children here; widen it with evidence.

## Related

#253 (blocked on this for acceptance) · #12 (@neo-opus-vega's unexplained orchestrator amber at `90d41ff`) · #237 · #73

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-09-19T11:52Z; all-state searches for `orchestrator healthcheck`, `unhealthy orchestrator`, `src/Neo.mjs healthcheck`, `zombie`, `init: true`, `PID 1`. Nothing equivalent (closed #270 covers empty backups publishing as newest, a different defect).

Origin Session ID: 6ecb7b5f-dc26-48a3-8e49-7232159377c1
Retrieval Hint: "orchestrator healthcheck ERR_MODULE_NOT_FOUND src/Neo.mjs Brain image zombie git init"

## Timeline

- 2026-09-19T11:52:24Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-19T11:52:25Z @neo-opus-ada added the `bug` label
- 2026-09-19T11:52:26Z @neo-opus-ada added the `ai` label
- 2026-09-19T11:52:26Z @neo-opus-ada added the `build` label
- 2026-09-19T11:52:26Z @neo-opus-ada added the `agent-os` label
- 2026-09-19T11:53:00Z @neo-opus-ada cross-referenced by #253
- 2026-09-19T11:58:50Z @neo-opus-ada cross-referenced by PR #374
- 2026-09-19T12:17:24Z @tobiu referenced in commit `11218d7` - "Merge pull request #374 from neomjs/ada/373-orchestrator-probe

fix(deploy): the orchestrator probe resolves in a Brain image, and init reaps its git children (#373)"
- 2026-09-19T12:17:25Z @tobiu closed this issue

