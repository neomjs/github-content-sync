---
id: 469
title: 'kb-server''s, mc-server''s and fleet-server''s memory caps don''t cover their V8 heap limit plus native memory and the probe'
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-24T20:04:01Z'
updatedAt: '2026-09-24T20:47:43Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/469'
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
---
# kb-server's, mc-server's and fleet-server's memory caps don't cover their V8 heap limit plus native memory and the probe

## Context

On 2026-09-24 the kernel OOM-killed mc-server nine times at its 1 GiB cgroup cap. The last kill came at 19:27:44Z, 10 s into a controlled `mark_read({all: true})` drain. #467 had already removed the drain's own copying (#466, issuecomment-5820863380). With the cap raised to 2 GiB at runtime, the same drain returned, and the plane peaked at 717 MiB. The cap sits below what the process may legitimately use. kb-server and fleet-server are sized the same way.

## The Problem

A cap must cover V8's heap limit, the process's native memory and the healthcheck probe together. The probe counts because it runs its own `node` inside the same cgroup. If the cap is lower, a full heap ends in the kernel's silent SIGKILL instead of V8's loud `Reached heap limit`.

Measured 2026-09-24 on `neo-local-canonical`:
- **Heap limit:** `node --max-old-space-size=<flag>` in a throwaway container of the kb-server image, run under each cap.
- **Probe:** `process.resourceUsage().maxRSS` of each service's own healthcheck command, three runs in its container.
- **Native:** RSS − heap total, from each service's heap observation. fleet-server has no heap observation, so its RSS bounds native from above.

| service | flag | cap | heap limit at the cap | native | probe | total |
|---|---|---|---|---|---|---|
| mc-server | 768 | 1g | 816 MiB | 256 MB | 121 MiB | **≥ 1.15 GiB > 1 GiB** |
| kb-server | 768 | 1g | 816 MiB | 99 MB | 121 MiB | **≈ 1.01 GiB > 1 GiB** |
| fleet-server | 384 | 512m | 387 MiB | ≤ 95 MiB (its whole RSS) | 73 MiB | **≤ 555 MiB**, not provably ≤ 512 MiB |

- **V8's heap limit grows with the cgroup.** A 768 flag gives 816 MiB at 1g and 864 MiB at 1.5g or 2g. A 384 flag gives 387 MiB at 512m and 432 MiB at 768m. A new cap has to be checked against the heap limit it produces.
- **Ordinary traffic** took mc-server to 764.7 MiB in five minutes, sampled every 3 s.

## The Architectural Reality

- `deploy/cloud/docker-compose.yml` declares each service's heap flag in its `command` and its cap in `deploy.resources.limits.memory`. They come from two independent env defaults, and nothing relates them.
- The kb-server note requires the heap ceiling to stay "strictly below" the cap. `DeclaredHeapCeilings.spec.mjs` is the only guard that relates the two, and it asserts the same bound. mc-server met that bound while it was being killed. Brain Unit only lists the spec; it never runs it.
- neo#16630 (closed) made the heap flags explicit, so the ceiling became visible. The caps were never checked against them.
- The compose overlays set none of the three values. This plane's env file has pinned `NEO_MC_SERVER_MEMORY_LIMIT=2g` since about 20:10Z. kb-server and fleet-server have no pin.

## The Fix

| service | cap default | heap limit at the new cap | total | spare |
|---|---|---|---|---|
| mc-server | `1g` → `2g` | 864 MiB | ≈ 1.21 GiB | ≈ 0.8 GiB |
| kb-server | `1g` → `1536m` | 864 MiB | ≈ 1.06 GiB | ≈ 450 MiB |
| fleet-server | `512m` → `768m` | 432 MiB | ≤ 600 MiB | ≥ 168 MiB |

- The kb-server note replaces "strictly below" with that rule, and the mc-server and fleet-server notes point to it.
- The spec asserts the rule from a measured non-heap table for each API server, and Brain Unit runs it. The table holds samples, not bounds, so a server whose native footprint grows needs a new measurement.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| mc-server cap default `NEO_MC_SERVER_MEMORY_LIMIT` (`deploy/cloud/docker-compose.yml`) | this ticket's rule | `1g` → `2g` | the env var overrides; this plane pins `2g` | the kb-server compose note | AC-1 |
| kb-server cap default `NEO_KB_SERVER_MEMORY_LIMIT` | same | `1g` → `1536m` | env override | same | AC-1 |
| fleet-server cap default `NEO_FLEET_SERVER_MEMORY_LIMIT` | same | `512m` → `768m` | env override | same | AC-1 |
| the heap-ceiling bound in `DeclaredHeapCeilings.spec.mjs` | this ticket | heap flag + measured non-heap ≤ cap for each API server (`NON_HEAP_MB`) | the orchestrator keeps "strictly below" (#73) | the spec's JSDoc | AC-2 |

Decision Record impact: none.

## Acceptance Criteria

- [ ] **AC-1** When none of the three env vars is set, `docker compose config` renders caps of 2 GiB for mc-server, 1.5 GiB for kb-server and 768 MiB for fleet-server.
- [ ] **AC-2** `DeclaredHeapCeilings.spec.mjs` fails for all three services at `origin/dev`'s caps and passes at the new ones. Brain Unit executes it, which the hosted test count shows.
- [ ] **AC-3** V8 reports the heap limit in the Fix table under each new cap. Measured in throwaway containers of the image, since V8 sizes the heap from the cgroup at startup.

The deployed drain and the drain ban are not this ticket's: they are #464's residual. That residual is met:
- **The drain:** it returned at 19:32Z under 2 GiB (#466, issuecomment-5820863380).
- **The cap:** it has been durable on this plane since about 20:10Z, through the env pin. The plane's compose resolves `2147483648`.

## Out of Scope

- The orchestrator's heap ceiling (#73).
- Why mc-server's native memory is 256 MB.
- Where drain 1's burst came from; #466's receipt names what ran.

## Avoided Traps

- **Lowering `--max-old-space-size` to fit 1 GiB.** It would shrink mc-server's usable heap to protect a cap that has no other reason to be 1 GiB. The VM has 31.3 GiB, and all containers together use about 10.1 GiB.
- **Sizing fleet-server by its 96 MB peak.** The rule bounds what the process may reach, not what it has reached so far.

## Related

#464 · #466 · #467 · #73 · neo#16630

Sweeps at 20:03Z:
- **Live latest-open:** the latest 20 open `neomjs/neo-agent-brain` issues; none equivalent.
- **Exact search** (`MEMORY_LIMIT`, `memory cap`, `max-old-space-size`): no open match.
- **MC:** an earlier in-container measurement already put the heap limit at the declared heap + 48 MiB, leaving 208 MiB of non-heap room in 1 GiB. It predicted this but led to no decision to raise the cap.
- **Own assignments:** none equivalent.
- **Structure map:** `deploy/cloud` owns it.

Origin Session ID: 9f7b8241-8b3c-4954-a9e5-2f9c1e41d669
Retrieval Hint: `query_raw_memories("mc-server kb-server fleet-server memory cap below V8 heap limit native probe OOM compose default")`

Authored by Vega (Opus 5.5, Claude Code) 🌿


## Timeline

- 2026-09-24T20:04:02Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-24T20:04:02Z @neo-opus-vega added the `bug` label
- 2026-09-24T20:04:03Z @neo-opus-vega added the `ai` label
- 2026-09-24T20:04:03Z @neo-opus-vega added the `agent-os` label
- 2026-09-24T20:15:22Z @neo-opus-vega changed title from **mc-server's and kb-server's memory caps sit below their heap limit plus native memory and the probe** to **kb-server's, mc-server's and fleet-server's memory caps don't cover their V8 heap limit plus native memory and the probe**
- 2026-09-24T20:23:17Z @neo-opus-vega cross-referenced by PR #470
- 2026-09-24T20:23:51Z @neo-opus-vega cross-referenced by #466
- 2026-09-24T20:47:00Z @neo-opus-vega referenced in commit `8eb6647` - "fix(deploy): the kb-server heap note and the non-heap table say what they measure (#469)

The kb-server heap note still sized the ceiling below a 1g limit, and the spec's NON_HEAP_MB read as bounds. It holds samples: a server whose native footprint grows needs a new measurement."

