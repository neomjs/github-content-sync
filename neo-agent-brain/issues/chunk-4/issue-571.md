---
id: 571
title: Every agent seat lives in one folder layout that Fleet provisions and launches into
state: OPEN
labels:
  - epic
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-09-27T09:46:57Z'
updatedAt: '2026-09-27T12:31:12Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/571'
author: neo-opus-ada
commentsCount: 0
parentIssue: null
subIssues:
  - '[x] 572 Fleet derives a seat''s clone and harness home under one agents root'
  - '[x] 574 Agent OS LaunchAgents copy the installer''s PATH and read a seat''s .env'
  - '[ ] 584 agents-root retirement: a stale instanceRoot parameter and no merged receipt'
  - '[x] 589 setRepo takes a validated GitHub slug and derives the clone URL itself'
  - '[x] 591 A seat''s first Start clones its repo with the seat''s own PAT, not the host''s credentials'
subIssuesCompleted: 4
subIssuesTotal: 5
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# Every agent seat lives in one folder layout that Fleet provisions and launches into

Terminal predicate: Every agent seat on this machine lives at `/Users/Shared/agents/<agent-id>/` — its clones at `<org>/<repo>`, its harness home at `harness/<type>` — Fleet provisions and launches into exactly those paths, and no machine daemon, shell arm or wake route resolves through a seat's folder or a pre-layout path.

## The Problem

The operator's direction (2026-09-27): one agents folder, not grouped by lab. Shell changes stay additive until harnesses start from FM; the old entries go only after that works.

Seats were added one at a time, and each got a root of its own (measured 2026-09-27):

| Seat | Root |
|---|---|
| `@neo-opus-ada` | `/Users/Shared/github/neomjs` |
| `@neo-opus-grace` | `/Users/Shared/claude/neomjs` |
| `@neo-gpt` | `/Users/Shared/codex/neomjs` |
| `@neo-opus-vega` | `/Users/Shared/opus-vega/neomjs` |
| `@neo-fable` | `/Users/Shared/fable/neomjs` |
| `@neo-fable-clio` | `/Users/Shared/clio/neomjs` |
| `@neo-gemini-pro` | `/Users/Shared/antigravity/neomjs` |
| `@neo-gpt-emmy`, `@neo-preview`, `@neo-kimi-iris`, `@neo-kimi-phoebe` | already `/Users/Shared/agents/<handle>/neomjs` |

Harness homes sit elsewhere again: `~/.claude-instances/<name>`, `~/.codex-app-instances/<name>`, `~/.codex`.

A seat cannot simply be moved, because much of its state is keyed by path:

- **Claude Code file-memory** is keyed by the cwd (`~/.claude/projects/<cwd-slug>/`; `learn/agentos/OwnAgentTeam.md` § Isolate Memory Correctly). A moved clone opens with empty memory unless its slug moves with it.
- `~/.claude.json` project entries and `~/.codex/config.toml` trust entries are keyed by path.
- `~/.zshenv` routes each seat's credentials through a cwd-prefix arm to `<seat>/neomjs/neo/.env`.
- **Each Claude app instance starts its MCP servers from the seat's engine clone:** `cd <seat>/neomjs/neo && node --env-file=<seat>/neomjs/neo/.env …`. That is 22 references across five configs (measured 2026-09-27, the same count as 2026-08-28). A running app rewrites its own config over an external edit, so a seat's references can only move while that seat is quit.
- **Brain processes load `<clone>/.env` through dotenv** (`ai/services.mjs`, the orchestrator entrypoints), so each clone carries its own copy of the seat's env (one or two per seat today).
- The wake route manifest names each harness's user-data dir.
- **Machine daemons still reach into seats.** #335 moved both Agent OS LaunchAgents' `WorkingDirectory` to the seat-neutral `/Users/Shared/agent-os/neo-agent-brain`. Both still carry a `PATH` captured from one interactive shell, with one seat's `neo/node_modules/.bin` on it. `host-edge` also reads its `.env` from inside another seat's clone: `DOTENV_CONFIG_PATH` → `…/neo-gpt-emmy/neomjs/neo/.neo-ai-secrets/agent-os-runtime/03035d1b…/.env`. Moving that seat takes the daemon's environment with it, which is the class #335 closed for the working directory. `com.neomjs.middleware-rebuild` runs from a seat's `middleware-v2` clone, and pulls, builds and deploys from that seat's engine clone (`../neo`). Its fetch races the seat's own, and whatever branch the seat has checked out is what it would ship. No run has completed since 2026-09-22 (defect-notes `9204448c7d6aaeca`, `03f31d552d6f05f3`; measured 2026-09-27).
- Leftovers: three seat trees still carry `.neo-ai-data/*` links to targets that no longer exist.

Fleet's own layout matches none of this. `deriveAgentRepoPath` and `deriveAgentInstanceHome` put a provisioned seat in two places, both beneath the plane's data root by default:
- clones: `<fleet.dataDir>/repos/<id>-<hash12>/<org-repo>-<hash12>`;
- harness home: `<fleet.instanceRoot>/<id>-<hash12>/<type>-<hash12>`.

Starting an existing seat from Fleet would therefore start it in a fresh clone, and a fresh cwd means empty file-memory. **Operator ruling (2026-09-27): Fleet Manager always holds a PAT, at least one per agent; the seats' `.env` files are a temporary stopgap until FM is usable.** An "external, no PAT" registration (neomjs/neo-agent-institution#280) contradicted it and is reverted by neomjs/neo-agent-institution#289.

**Why an Epic:** the change spans three places, and the operator fixed the order across them:
- Brain source: Fleet's path derivation and its config;
- the machine: shell arms, LaunchAgents, per-seat moves;
- the Institution: starting a seat from FM.

No single PR can carry it.

## Intended shape

1. **One root, one layout, keyed by agent id.** Clones go to `/Users/Shared/agents/<agent-id>/<org>/<repo>` and the harness home to `/Users/Shared/agents/<agent-id>/harness/<type>`, with no lab level. The key stays the Fleet agent id, never the GitHub login: per `deriveAgentInstanceHome`'s rule, two agents may share a login but never a home. For every current seat, the id is its handle.
2. **Fleet derives the path a person would make.** Segments are readable, and validated rather than hashed. An id or slug outside a safe charset is refused instead of sanitized, so two raw values cannot collide and no hash is needed.
   - The root is one declared AiConfig leaf and not a plane member. A seat folder holds working trees and path-keyed memory that must outlive any plane (ADR-0019 §10.9).
   - The two entrypoints that each compute `path.join(fleet.dataDir, 'repos')` today read that leaf instead.
   - If Fleet control runs in a container (#83), the host root and the container mount are two named contracts, never one value (ADR-0019 §10.9's paired rule).
3. **FM starts every seat, with its PAT, in its own clone.** Once Fleet's path equals the seat's path, FM starts a seat where its memory already is. That is what retires the seats' `.env` files.
4. **Machine surfaces stop resolving through seats.** The Agent OS LaunchAgents get a seat-neutral `PATH` and a seat-neutral `.env`, finishing #335. `middleware-rebuild` gets checkouts of its own for `middleware-v2` and the engine; its plist lives in that private repo.
5. **Migration order:**
   - one generic `/Users/Shared/agents/*` shell arm, added after the existing arms, so running sessions keep their env;
   - a pilot on seats not running this week (proposed: Clio, Mnemosyne, Gemini). Each move carries along the Claude project slug, the `~/.claude.json` and Codex trust entries, the harness data, the app's MCP config (re-pointed with the app quit), the per-clone `.env` copies and the wake route;
   - the gate: FM starts a piloted seat in the new layout;
   - live seats one at a time, each asked on A2A first;
   - old arms and leftovers go last.

   Every machine change waits for the operator's explicit OK.

## Constraints

- Harness homes hold credentials and sit under the operator's home today. `/Users/Shared` is shared by design, so the layout must not widen who can read them.
- A move re-keys path-keyed memory and never orphans it. #142 owns the reconciliation when a seat is removed.

## Out of Scope

- The canonical plane roster and registry: #52 (S4), #53 (S3).
- Other machines and cloud seats.

## Related

#335 · #142 · #84 · #83 · neomjs/neo-agent-institution#289 · neomjs/neo-agent-institution#7 · `learn/agentos/OwnAgentTeam.md`

Prior art: Clio began the same move on 2026-08-28, with a seat-root `.env` template and a zshenv replacement. Only her own seat-root `.env` landed; the zshenv arms still route to the in-clone file. Vega's census of the MCP-config references came out of that work.

Structure map: `npm run ai:structure-map -- --files --loc` at `4bc885b`. Owning code:
- `ai/services/fleet/`: `deriveAgentRepoPath`, `deriveAgentInstanceHome`, `devFleetServer`, `fleetServer`;
- the `fleet` leaves in `ai/configBase.mjs`;
- the LaunchAgent prescriptions in `ai/scripts/lifecycle/local-agent-os/`.

Epic-layer sweep (arm v): open epics here (34), in the Institution (7) and in the engine (25). I read their predicates where present and grepped their bodies for layout terms: 0 hits. None names this outcome; the nearest are #84 (machine cut-over to the Docker plane), #83 (containerized Fleet control) and neomjs/neo-agent-institution#7 (Electron shell).

Other sweeps:
- Issue search: #335 (closed; this epic finishes its `PATH`/`.env` remainder) and #142.
- Latest 20 open issues here: no equivalent.
- A2A: no layout claim.

Origin Session ID: f3d50317-fe3b-4773-b4ac-db05e1fa6812
Retrieval Hint: `query_raw_memories("seat folder layout /Users/Shared/agents one root Fleet derives plain paths zshenv additive arm")`




## Timeline

- 2026-09-27T09:46:57Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-27T09:46:59Z @neo-opus-ada added the `epic` label
- 2026-09-27T09:46:59Z @neo-opus-ada added the `ai` label
- 2026-09-27T09:46:59Z @neo-opus-ada added the `architecture` label
- 2026-09-27T09:46:59Z @neo-opus-ada added the `agent-os` label
- 2026-09-27T10:01:33Z @neo-opus-ada cross-referenced by PR #281
- 2026-09-27T10:30:23Z @neo-opus-ada cross-referenced by #572
- 2026-09-27T10:30:28Z @neo-opus-ada added sub-issue #572
- 2026-09-27T11:04:26Z @neo-opus-ada cross-referenced by PR #573
- 2026-09-27T11:49:22Z @neo-opus-ada cross-referenced by #574
- 2026-09-27T11:49:34Z @neo-opus-ada added sub-issue #574
- 2026-09-27T12:02:00Z @neo-opus-ada cross-referenced by PR #575
- 2026-09-27T12:26:56Z @neo-opus-ada cross-referenced by #289
- 2026-09-27T12:59:53Z @neo-opus-ada cross-referenced by PR #577
- 2026-09-27T14:35:31Z @neo-preview added sub-issue #584
- 2026-09-27T15:10:00Z @neo-opus-ada cross-referenced by #589
- 2026-09-27T15:10:03Z @neo-opus-ada added sub-issue #589
- 2026-09-27T15:13:33Z @neo-opus-ada cross-referenced by PR #590
- 2026-09-27T15:46:21Z @neo-opus-ada cross-referenced by #591
- 2026-09-27T15:46:24Z @neo-opus-ada added sub-issue #591
- 2026-09-27T15:50:52Z @neo-opus-ada cross-referenced by PR #592
- 2026-09-27T16:15:16Z @neo-preview cross-referenced by PR #588

