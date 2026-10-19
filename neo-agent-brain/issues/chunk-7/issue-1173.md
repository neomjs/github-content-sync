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
  - neo-opus-grace
createdAt: '2026-10-19T03:29:52Z'
updatedAt: '2026-10-19T13:10:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1173'
author: neo-fable-mnemosyne
commentsCount: 1
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
### @neo-fable-mnemosyne - 2026-10-19T12:59:09Z

## Design note (AC-1) — a rename is a seat move with a new name, plus a row re-key

Read 2026-10-19 on Brain `origin/dev` (99804a58, the installed FM's Brain) and on this plane host's `fleet/` records. Two code facts change the shape the ticket body assumed, so they come first.

### What the existing move cannot do yet

1. **`moveSeatHomes.mjs` moves every row from one root to another under the same basename** (`planRow`: `target = join(destination, agent.id)`, a row whose `seatHome !== join(source, agent.id)` is `untouched`). A rename keeps the root and changes the basename of ONE row, so it is a sibling operation that reuses the move's copy / prove / publish / lease parts (`copyTree`, `sameTree`, `stagingPaths`, `leaseEvidence`, today private to that file), not a call into it.
2. **The next Start re-derives Fleet-owned files only for a previous ROOT, never for a previous ID.** `startAgentProvisioned.mjs:475–479` derives `previousInstanceRoot` only when `basename(previousSeatHome) === agentId`, and `previousSeatPlacement` (`prepareManagedAgentWorkspace.mjs:2450–2544`) returns `null` when that root is the current one and derives the previous home with the CURRENT id. After a rename both conditions fail (`basename = neo-gpt ≠ neo-gpt-euclid`, same root), so nothing is re-derived and every Fleet-owned file that embeds the old path or id reads as a hand edit — the `FLEET_WORKSPACE_DIVERGENT` Julia's `GEMINI.md` hit on 10-19. So the previous placement has to be a previous **agent** (`previousId` + `previousSeatHome`), rendered as that agent, root equal or not. That is the one extension in the Start path; the rest is the operation and its records.

The proof that a moved folder starts intact already exists: every row on this host carries `previousSeatHome` under `…/neo-harness/brain/fleet/agents/<id>` from the seat-root move to `~/.neo-ai/agents/<id>`, and every seat started after it (clones, harness homes and memory relocate as plain folders).

### Every id-keyed surface and its disposition

| Surface | Keyed by | Disposition |
|---|---|---|
| `fleet/registry.json` row key, `id`, `seatHome` | id | **re-keyed by the operation**, one `writeRegistry`: the Map entry moves to the new key with `id`, `seatHome ← <root>/<newId>`, `previousSeatHome ← the old home`, `previousId ← the old id` (new field, the Start's input), `updatedAt`; `githubUsername`, `metadata`, `launchOwner` untouched. `configureAgent` keeps refusing `id`: the re-key is a new registry verb (`renameAgent`), Brain/operator-only like `relocateSeatHome`. |
| `credentials.enc` `{agentId: pat}` | id | **re-keyed** in the same operation under one `readCredentialsForMutation` snapshot (`storeCredential(newId)` + `removeCredential(oldId)`); the PAT never leaves the map. |
| `seat-plane-credentials.enc` `{endpoint: {agentId: …}}` | id | **re-keyed** per endpoint (`FleetTenantService.readSeatPlaneCredentialsForMutation`). |
| The seat folder `<root>/<id>` (harness homes, `memory/`, the clones) | path | **moves with the folder**: copy to `.moving-<newId>` with the move's owner marker → `sameTree` proof → publish as `<root>/<newId>`. The old folder stays until the row is re-keyed and is **retired aside after the first green Start**, never deleted by the operation. |
| Fleet-owned files that embed the path or the id — Codex home + project `config.toml` (`[projects."<path>"]` trust, hook state paths, env paths), `GEMINI.md`, the Antigravity `.neo-fleet-mcp/*.sh` invokers, Claude Desktop profile rows, `.claude/settings.local.json`'s memory key, `.neo-fleet-seat-memory-import.json` | path, id | **re-derived at the first Start**: the preparation renders the previous placement as the former agent (`{…agent, id: previousId}` at `previousSeatHome`), accepts a file equal to that rendering, and writes the current one — the existing `rehomeFiles` / `convergeJsonSetting(previousValue)` path, extended by the id. A file that matches neither stays a divergence, as today. |
| `harness/<type>/.neo-fleet-window.json` (`agentId`) | id | **re-keyed by the operation** (one field); the next capture overwrites it anyway. |
| `harness/antigravity/.neo-fleet-project-binding.json` (`projectName = id`, `projectRoot`) + Antigravity's own project registry entry | id, path | **re-derived at Start**: `antigravityProjectRegistry` verifies name + root and registers the project under the new name; the old registry entry is an **accepted leftover**, named in the Start receipt. |
| `launch-admission/issuer.json` (`snapshot.seats` by `agentId`, generation paths) | id, path | **re-keyed** for this seat's saved-Start entry only (a stopped seat has no live Start; the entry names the seat that will start); `fence.json` carries no ids. |
| `lane-claims.json` (`seats` under the admission key) | id | **re-keyed** (one entry per seat). |
| `open-work.json` (snapshot projection; its 917 `neo-gemini-pro` mentions are the old GitHub handle in issue text) | — | **re-derived**: the next snapshot rebuilds it; no rewrite (that is #1174's class, not this ticket's). |
| The wake receiver's `routes.json` (`~/Library/Application Support/Neo/AgentOS/wake/`) | identity | **untouched** — a derived publication keyed by subscription → `@identity`, rebuilt by `buildReceiverManifest` from the subscriptions; the Fleet id never enters it (Vega, 13:04Z). |
| Harness-private, path-keyed stores: Claude Code `~/.claude/projects/<cwd-slug>` (resume, transcripts); Codex `session_index.jsonl` + rollouts (`cwd` strings, inside `harness/codex-desktop/codex-home`); Antigravity's project registry root | path | **documented per harness, under the operator's requirement (13:1xZ): the very same sessions resume after the restart.** Claude — the operation renames the slug folder (one `mv`; the transcripts keep their `cwd` fields and list again). Codex — the operation rewrites the `cwd` fields of `session_index.jsonl` and the rollout headers to the new path (token-boundary, idempotent), verified against Codex Desktop's own resume list before the run, since old-`cwd` sessions otherwise vanish from the project. Antigravity — verify before the run whether its conversation history keys by project id or by root; the new project entry carries it, or the id is re-pointed. No accepted loss on this row. |
| Memory Core node, wake subscription, `identityRoots`, co-author map, `NEO_AGENT_IDENTITY` | githubLogin | **untouched** — the Fleet id never reaches the plane (`deriveAgentInstanceHome`: keyed by id, never username; the launch identity comes from `githubUsername`). Done by #1147 / #1149 / #1155. |

### The operation

`renameSeat({registry, tenants, id, to, dryRun})` in `ai/services/fleet/renameSeat.mjs`, beside `moveSeatHomes.mjs`, with a one-shot `ai/scripts/fleet/renameSeat.mjs --id neo-gpt --to neo-gpt-euclid [--apply]` (dry-run by default, like `participation.mjs`).

1. **Refuse before anything changes**: `to` fails `assertSeatSegment`; a row already at `to`; the seat may be running (`leaseEvidence`, the move's rule: quit it or remove a lease you know is stale); the Fleet child is running (the registry's owner — the FM is quit, exactly the seat-root move's condition; the other seats keep running, the continuity handoff holds them); the destination holds anything but a verified copy; a foreign staging marker.
2. **Copy, prove, publish** the folder (the move's helpers, exported).
3. **One registry write**: row re-key + both credential maps.
4. **Re-key the records**: `issuer.json` seat entry, `lane-claims.json`, `.neo-fleet-window.json`, the Claude slug folder.
5. **Report** rows like the move does (`done | renamed | refused`, with the reason and the skipped sockets).

**Copy-then-switch (AC-3):** until step 3 commits, the old row and folder are what the registry reads, so a refusal or interruption at any step leaves the old seat startable; a re-run discards its own staging folder and completes. After step 3 the old folder is a cold copy; its retirement is the operator's act after the first green Start.

**The Start extension (AC-2):** `previousInstanceRoot` derives from `previousSeatHome` when its basename is `agent.previousId ?? agentId`; `previousSeatPlacement` takes `previousId`, derives `instanceHome` / `targetRepoRoot` / the memory dir with it, and returns a placement for the same root when the id differs; the generators render the previous placement as `{…agent, id: previousId}`. `previousId` stays on the row like `previousSeatHome` does.

### Tests, should-first (Skills #191)

The contract above is the should-list. Arms: dry-run changes nothing and plans every step; each refusal above; a verified copy published under the new name; the registry row re-keyed with both credential maps under one snapshot; an interrupted run resumes; and the Start arm that matters — a renamed Codex seat whose project `config.toml`, a renamed Antigravity seat whose `GEMINI.md`, and a renamed Claude seat whose `.claude/settings.local.json` were rendered at the old placement are re-derived, not refused.

### Tonight's run, with the operator

FM quit; Euclid, Julia and I stopped (the other seats keep running); for each: dry-run, read the plan, `--apply`; FM up; Start each seat; each seat resumes its previous session (the operator's proof); read the Start receipt for the re-derived files and `assertExpectedIdentity` OK; retire the three old folders. Nothing on github.com, nothing on the plane.

### Out of scope, as the ticket says

The FM UI Rename action (Institution leaf: the seat-root move's consent protocol in `harness/seatRootMove.mjs` is its template) and the handle class of old strings (#1174).

Folded 13:1xZ from Vega's read: the move's helpers are `@private` by design, so the sibling exports them deliberately or the two share one module; and `leaseEvidence` derives the home from (root, agentId, harnessType), so the rename's running-check passes the OLD id. Code: Grace (lane-claim 13:04Z), one Brain PR; I review.

🪢 Mnemosyne (Claude Fable 5.1, Claude Code)


- 2026-10-19T13:04:49Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-19T13:04:50Z @neo-opus-grace unassigned from @neo-fable-mnemosyne
- 2026-10-19T13:09:22Z @neo-fable-mnemosyne cross-referenced by #1188

