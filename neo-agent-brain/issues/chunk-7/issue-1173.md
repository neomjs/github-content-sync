---
id: 1173
title: 'Rename a seat''s Fleet id: folder, registry row and every id-keyed record follow'
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-fable-mnemosyne
createdAt: '2026-10-19T03:29:52Z'
updatedAt: '2026-10-19T03:29:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1173'
author: neo-fable-mnemosyne
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
# Rename a seat's Fleet id: folder, registry row and every id-keyed record follow

## Context

Operator direction, 2026-10-19 05:2x Berlin, after the handle renames (#1147, #1149, #1155) and tonight's FM update: the three renamed seats still carry their old Fleet ids — `neo-gpt` (Euclid), `neo-gemini-pro` (Julia), `neo-fable` (Mnemosyne) — and the ids follow the handles on 2026-10-20. Banked here as the plan-of-record; design before build.

Design authority: the operator, 2026-10-19 ("this one needs to happen tomorrow"; expected friction: repo folders and data dirs move).

## The Problem

A seat's Fleet id is its folder name under the agents root and the key of every id-keyed record. The handle rename moved the login (registry `githubUsername`, roots, graph, launch env) but not the id, so Euclid's row reads `id: neo-gpt` beside `githubUsername: neo-gpt-euclid`, his home is `<agentsRoot>/neo-gpt`, and every Fleet-rendered file embeds that path. Renaming by hand breaks the seat the way Julia's `GEMINI.md` did tonight: a Fleet-owned file rendered at the old path reads as a hand edit at the next Start and refuses `FLEET_WORKSPACE_DIVERGENT`.

## The Architectural Reality

Observed 2026-10-19 03:3xZ on the plane host (counts are lines naming an old id):

- Registry `fleet/registry.json`: the row key is the id; `seatHome` and `previousSeatHome` carry the folder; `credentials.enc` and `seat-plane-credentials.enc` resolve by id (`registry.resolveCredential(agentId)`); `FleetRegistryService.configureAgent` allows neither `id` nor `githubUsername` changes (Vega's defect-note of 00:44Z).
- Seat folder `<agentsRoot>/<id>`: `harness/<type>/` (the instance home), `memory/`, the clones under `neomjs/`; `deriveAgentInstanceHome` and `deriveAgentMemoryDir` derive from `agent.id`.
- Fleet-owned files embedding the seat path: the Codex home and project `config.toml` (the `[projects."<path>"]` trust block, `hooks.state` paths, env paths), `.neo-fleet-seat-memory-import.json`, `GEMINI.md`, the Antigravity `.neo-fleet-mcp/*.sh` invokers, Claude Desktop profile rows. `.neo-fleet-window.json` carries `agentId`; `.neo-fleet-project-binding.json` and the Antigravity project registry carry `projectName = agent.id`.
- Id-keyed Fleet records: `launch-admission/issuer.json` (saved Starts: `agentId`; 1), `lane-claims.json` (1), `open-work.json` (170 mentions, a projection), the wake receiver's `routes.json` (3 route owners, 1 seat path).
- Harness-private, path-keyed stores: Claude Code's history under `~/.claude/projects/<cwd-slug>` (resume breaks when the cwd moves), Codex's `session_index.jsonl` and rollouts (cwd), Antigravity's project registry (root path).
- Existing machinery: `moveSeatHomes.mjs` copies seat homes to a new agents root and records `previousSeatHome`; the next provisioned Start re-derives every Fleet-owned file rendered at the previous home (`previousInstanceRoot` → `previousSeatPlacement` in `prepareManagedAgentWorkspace.mjs`). An id rename is that move with a different folder name plus a row re-key.

## The Fix

One Brain operation, run with the seat stopped (or the FM quit): re-key the registry row (`id`, `seatHome`; `previousSeatHome` ← the old home), move the folder under the new name, rewrite the id-keyed Fleet records, and let the next provisioned Start re-derive the Fleet-owned files through the existing previous-placement path. The harness-private history stores either move with the folder, documented per harness, or are named accepted losses. The design pass on this ticket fixes the shape before code; the FM UI leg (a Rename action) is a separate Institution leaf.

## Acceptance Criteria

- [ ] A design note on this ticket lists every id-keyed surface with its disposition (re-keyed / re-derived at Start / moves with the folder / accepted loss) before any code.
- [ ] The three seats start under their new ids on the installed FM with memory, clones, window records and credentials intact, without a `FLEET_WORKSPACE_DIVERGENT` refusal.
- [ ] A rename refused half-way leaves the old seat startable (copy-then-switch, as the root move does).
- [ ] [L3-deferred — operator handoff needed] Installed: the three renames run on the plane host on 2026-10-20 with the operator. Residual-Owner: neomjs/neo-agent-institution#732.

## Out of Scope

- The FM UI Rename action (Institution leaf).
- Handle renames (done: #1147, #1149, #1155).

## Related

#1147, #1149, #1155, #1167, #1170, neomjs/neo-agent-institution#732.

Live latest-open sweep: the 20 latest open Brain issues at 03:3xZ, no equivalent. A2A claim sweep of the last hour: none. Memory Core semantic sweep: unavailable (embedding probe timeout), attested.

Origin Session ID: 830c2630-dbf5-47fb-9170-4cb6dee2b08c
Retrieval Hint: "seat Fleet id rename folder registry re-key previousSeatHome"

## Timeline

- 2026-10-19T03:29:53Z @neo-fable-mnemosyne assigned to @neo-fable-mnemosyne
- 2026-10-19T03:29:54Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-10-19T03:29:54Z @neo-fable-mnemosyne added the `ai` label
- 2026-10-19T03:29:54Z @neo-fable-mnemosyne added the `architecture` label
- 2026-10-19T03:29:54Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-10-19T03:37:14Z @neo-fable-mnemosyne cross-referenced by #1174

