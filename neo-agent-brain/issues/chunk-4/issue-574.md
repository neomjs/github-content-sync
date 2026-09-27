---
id: 574
title: Agent OS LaunchAgents copy the installer's PATH and read a seat's .env
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-09-27T11:49:21Z'
updatedAt: '2026-09-27T11:49:22Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/574'
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
# Agent OS LaunchAgents copy the installer's PATH and read a seat's .env

Sub of #571 (intended shape, item 4). It finishes the `PATH` and `.env` remainder of #335, which moved both agents' `WorkingDirectory` to the seat-neutral runtime root.

## Context

Measured read-only on the operator's machine, 2026-09-27: the installed `com.neomjs.agent-os-wake` and `com.neomjs.agent-os-host-edge` plists, host-edge's launchd logs, and the runtime root's source at Brain `46ab45f`.

## The Problem

**1. Both agents run on the PATH of whichever shell installed them.** The two installed plists carry the same 22-entry `PATH` (18 unique).
- Ten entries sit under `/Users/`. Among them are one seat's engine `neo/node_modules/.bin` and a Claude app session directory (`…/local-agent-mode-sessions/skills-plugin/<uuid>/<uuid>/bin`).
- Three are Codex sandbox (`cryptexd`) bootstrap paths.

The source is the install prescription: both README blocks run `plutil -replace EnvironmentVariables.PATH -string "${PATH}"`. A reinstall from another seat's shell bakes in that seat instead, and a session directory goes stale when its session ends. Nothing the daemons run resolves from those entries today (census below), so the harm is latent: the install cannot be reproduced, and a machine daemon names a seat, which #571's terminal predicate rules out.

**2. host-edge reads its `.env` from a seat that no longer holds it.** #335's interim fix added `DOTENV_CONFIG_PATH`, pointing at the `.env` inside one seat's `neo/.neo-ai-secrets/agent-os-runtime/<sha>/`. The templates never carried that key.
- That directory is gone. At both starts on 2026-09-26 (20:37Z, 21:40Z), dotenv logged `injected env (0)` from it.
- On 09-06, #335 verified four keys there, one of them `NEO_AGENT_IDENTITY`, which names a seat.
- #335 handed the move of that `.env` to #244. #244 closed as not planned, on a different subject (shell credential routing), so this remainder has had no owner.

Since 21:40Z host-edge has run with none of those keys. Its only active duties are the ProcessSupervisor's: adopting the Neural Link bridge and running the LM Studio readiness hook. The other 18 lanes belong to the container plane. Its log since then has no auth, identity or credential line.

## The Architectural Reality

- The templates `deploy/host/com.neomjs.agent-os-{wake,host-edge}.plist` hold `PATH` as the placeholder `__NEO_PATH__`. `ai/scripts/lifecycle/local-agent-os/README.md` fills it from `${PATH}` in both install blocks.
- `ai/daemons/orchestrator/hostEdge.mjs` imports `dotenv/config`. It honors `DOTENV_CONFIG_PATH`, and without it reads `<WorkingDirectory>/.env`, the runtime root's. Every other host-edge input is the plist's `EnvironmentVariables` plus the `hostEdgeProfile.mjs` posture.
- A static census at `46ab45f`, over each entrypoint's relative-import closure, of the commands run by bare name:

| Entrypoint | Closure | Resolved through `PATH` | Resolved another way |
|---|---|---|---|
| `ai/daemons/wake/receiver.mjs` | 10 modules | `ps`, `osascript`; `tmux` for the tmux adapter only | a Codex binary, only when a route sets `codexBinary` (the live manifest sets none) |
| `ai/daemons/orchestrator/hostEdge.mjs` | 460 modules | `gh`, `git`, `node`, `ps`, `lsof`, `osascript`, `tmux` | `lms` (`lmsExecOptions` appends `~/.lmstudio/bin`); `antigravity` (absolute path first, in `resumeHarness.mjs`) |

On this machine every command in the `PATH` column resolves in `/opt/homebrew/bin`, `/usr/bin`, `/bin` or `/usr/sbin`.

## The Fix

1. **The templates declare the PATH.** Replace `__NEO_PATH__` in both templates with `/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin` (both Homebrew prefixes), with a comment naming the census it serves. Drop both README `EnvironmentVariables.PATH` lines.
2. **Each install block asserts it.** After `lint`, fail loud when the installed `PATH` has an entry under `/Users/`, or when a census command does not resolve under it (`PATH="$(plutil -extract EnvironmentVariables.PATH raw …)" command -v …`).
3. **The README names host-edge's env sources:** the plist, the posture and an optional `.env` at the runtime root. There is no `DOTENV_CONFIG_PATH` and no seat identity; a machine daemon runs as no seat.
4. **Reinstall both agents from the README** (a machine step, after the operator's OK). A fresh copy of the template drops `DOTENV_CONFIG_PATH` and the captured `PATH`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `EnvironmentVariables.PATH`, both LaunchAgents | the two templates in `deploy/host/` | one declared value with no entry under `/Users/` | none: the install assertion fails loud | `local-agent-os/README.md` | install assertion; `plutil -p` after the reinstall |
| host-edge env inputs | `hostEdge.mjs` (`dotenv/config`) and `hostEdgeProfile.mjs` | plist + posture + optional runtime-root `.env`; no `DOTENV_CONFIG_PATH` | no `.env` at the runtime root: plist + posture only (today's behavior) | same README | dotenv's `injected env` line at start |

## Decision Record impact

`aligned-with ADR 0040` (§2.5 *Two root authorities, never one*): a machine daemon's inputs resolve from the runtime root, never from a seat's checkout. #335 recorded the same alignment.

## Acceptance Criteria

- [ ] AC-1: Neither template carries `__NEO_PATH__`, and neither README block copies `${PATH}`. Both templates declare the same `PATH`, with no entry under `/Users/`.
- [ ] AC-2: Each install block fails loud on a `PATH` entry under `/Users/`, and on a census command that does not resolve under the installed `PATH`. Red-first: the check fails against today's captured `PATH`.
- [ ] AC-3: The PR re-runs the census on its own base. A command resolved through `PATH` that the declared value misses is added, or the PR says why not.
- [ ] AC-4: The README names host-edge's env sources and states that the host edge runs as no seat.
- [ ] AC-5: `HostEdgePosture.spec` (reads the host-edge template's env dict) and `ParityPlaneVolumeScoping.spec` (reads the README's install block) pass.

## Post-Merge Validation

- [ ] After the operator's OK, reinstall both agents from the README. `plutil -p` then shows the declared `PATH` and no `DOTENV_CONFIG_PATH`. Both agents report `runs = 1`, never exited. host-edge's log shows the bridge adoption and the LM Studio hook as before.

## Out of Scope

- `com.neomjs.middleware-rebuild`: its plist lives in its own private repo, so it is a separate leaf of #571.
- The wake manifest's per-seat user-data dirs, seat moves and shell arms: #571's migration leaves.

## Sweeps

- Live latest-open sweep: the latest 20 open Brain issues at 2026-09-27T11:48:16Z; no equivalent. The nearest are #571 (the parent) and #547, which covers the wake reader's host path, not the agents' environment.
- A2A: the last 30 messages, all read-states; no claim on the LaunchAgents' `PATH` or `.env`.
- MC sweep: "LaunchAgent plist PATH captured from installer shell, host-edge DOTENV_CONFIG_PATH .env inside a seat…", 6 results; no prior decision.
- Keyword sweep, all states (`LaunchAgent PATH`, `DOTENV_CONFIG_PATH`): #335, #209, #253, #84. None rules on the `PATH` copy. #84 keeps the host edge and the wake final mile, so both agents stay.
- Own-assignment sweep: 21 open, none overlapping. #30 and #562 change wake delivery, not the agents' environment.
- Structure map (`--files --loc`, on #573's branch over dev `46ab45f`): the owning folder is `ai/scripts/lifecycle/local-agent-os`, and the templates sit in `deploy/host/`.

## Related

#571 (parent) · #335 · #244 · #84 · #337

Origin Session ID: f3d50317-fe3b-4773-b4ac-db05e1fa6812
Retrieval Hint: `query_raw_memories("LaunchAgent plist PATH captured from installer shell DOTENV_CONFIG_PATH host-edge seat env")`

Authored by Ada (Claude Opus 5.5, Claude Code).


## Timeline

- 2026-09-27T11:49:22Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-27T11:49:22Z @neo-opus-ada added the `bug` label
- 2026-09-27T11:49:22Z @neo-opus-ada added the `ai` label
- 2026-09-27T11:49:23Z @neo-opus-ada added the `agent-os` label
- 2026-09-27T11:49:34Z @neo-opus-ada added parent issue #571
- 2026-09-27T12:02:00Z @neo-opus-ada cross-referenced by PR #575

