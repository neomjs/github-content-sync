---
id: 1149
title: 'Run the identity rename against the plane''s volumes, writers stopped'
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-18T20:48:56Z'
updatedAt: '2026-10-19T00:44:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1149'
author: neo-opus-vega
commentsCount: 2
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1147 The first seats'' handles carry their Social Names: neo-gpt-euclid, neo-gemini-julia, neo-fable-mnemosyne'
  - '[x] 1146 renameAgentIdentities rewrites handles by substring: prefix pairs chain and sibling seats collide'
blocking: []
---
# Run the identity rename against the plane's volumes, writers stopped

## Context

neomjs/neo-agent-brain#1147 renames three seat handles in tonight's operator-led window (2026-10-18) and hands this ticket the one-shot migration run against the plane's volumes (its window steps 5 and 7). neomjs/neo-agent-brain#1146 fixes the script's rewriter. I rehearsed the run read-only against the live plane at ~20:45Z, before either PR merged.

Sweeps: live latest-open sweep at 2026-10-18T20:47Z, latest 20 open issues; no equivalent (#1146 and #1147 are the siblings). A2A claims: Mnemosyne's `bc62705c` names this slot mine, after my lane split `6ff370b3`. Memory Core: the 2026-06-05 apply (graph backup first, harness restart after) and `#13827` (a rename can strand memories under the old RLS scope; this script rewrites `user_id`, and #1147's recall AC checks it). Structure map: no new file.

## The Problem

The script edits one SQLite file and two Chroma collections. On the canonical local plane:
- The graph lives in the `shared-sqlite-data` volume, which three services mount: `mc-server`, `orchestrator` and `kb-server`. A run beside them races their writes, and a run from the host can't reach the volume.
- Chroma is the `chroma` service on the plane network, so the run needs that network too.
- The live project wasn't started with the README's command. Its containers carry a third compose file (the 2026-09-23 cut fragment that sets the orchestrator's sync-lane env) and a different env file. A `compose` call copied from the README would resolve a different configuration and drop that env.
- On 2026-06-16 a Memory Core restarted before the migration re-seeded the old identity node from stale roots.

## The Architectural Reality

Measured 2026-10-18 ~20:45Z, plane at `d6a905dc`:
- **Invocation.** The containers' `com.docker.compose.project.config_files`, `.environment_file` and `.working_dir` labels give the exact `docker compose` call. With it, `config --services` lists the six services.
- **One-shot.** `compose run --rm --no-deps -T mc-server node ai/scripts/migrations/renameAgentIdentities.mjs` resolves the graph to `/app/.neo-ai-data/sqlite/memory-core-graph.sqlite` (the shared volume) and Chroma to `chroma` (`NEO_CHROMA_HOST`). The image workdir is `/app`.
- **Rehearsal.** Dry runs with today's June pairs while the plane was live; no writes; each exited 0, about 10 s in total:

| Store | Scanned | Would change |
|---|---:|---|
| graph | — | 0 node/edge renames; 11 node + 6 edge metadata rows |
| `neo-agent-sessions` | 4,697 | 146 |
| `neo-agent-memory` | 48,671 | 0 |

  So June-era handles remain in 17 graph metadata rows and 146 session records. Whether #1146 keeps the June pairs decides whether tonight's apply finishes them.
- **Fleet rows (#1147 step 6).** No path changes `githubUsername`: `FleetRegistryService.updateAgent` patches `metadata` and `modelProvider` only (`ai/services/fleet/FleetRegistryService.mjs:612`).
- **FM install (#1147 step 4).** The continuity handoff refuses when any running seat has an unprovable server (#1143), and a Codex seat's Neural Link proof lapses an hour after its Start (#1145).

## The Fix: the window recipe

Preconditions: #1146 and #1147 merged. Plane images built at the merge SHA (`NEO_REVISION` resolved, per `ai/scripts/lifecycle/local-agent-os/README.md`) but not yet recreated.

1. Build `DC` from the labels above. `"${DC[@]}" config --services` lists six services.
2. Stop the writers and keep Chroma up: `"${DC[@]}" stop orchestrator mc-server kb-server`.
3. Copy `memory-core-graph.sqlite` (and its `-wal`/`-shm`) to a dated name in the same volume through a one-shot.
4. Dry run on the new image: `"${DC[@]}" run --rm --no-deps -T mc-server node ai/scripts/migrations/renameAgentIdentities.mjs`. Post the summary here. The three new pairs must show work.
5. Re-run with `--apply` and post the summary.
6. Fleet rows, as a stopgap until a verb exists: while the FM is quit for its install, back up the registry and set `githubUsername` on the three rows. `id` and the seat home stay unchanged.
7. `"${DC[@]}" up -d --wait`. `cat /app/.neo-revision` in `mc-server` must equal the merge SHA.
8. A fresh dry run reports zero work. If it doesn't (the 06-16 re-seed), stop `mc-server`, run `--graph-only --apply`, start it, then dry-run again.

FM install timing, a stopgap until #1143 and #1145 land: install while no running Codex seat is more than an hour past its Start, or restart those seats first.

## Acceptance Criteria

- [x] Rehearsed read-only on the live plane before the window (table above).
- [x] In the window: the dry-run and apply summaries are posted here, with the writers stopped, Chroma up and the graph backup named.
- [x] `/app/.neo-revision` in `mc-server` equals the #1147 merge SHA.
- [ ] The three Fleet rows carry the new logins, and the missing verb is filed as a defect-note.
- [x] The post-start dry run reports zero work, or the graph-only re-run reaches zero.
- [ ] #1147's boot ACs pass (receipts there).

## Out of Scope

- The rewriter fix (#1146), and the roots and consumers (#1147).
- A product verb for login renames.
- Fixes for #1143 and #1145.

## Avoided Traps

- **Copying the README's compose command:** it resolves a different configuration and drops the fragment's orchestrator env.
- **`docker exec` into the live `mc-server`:** it races the running writer, and the old image lacks #1146.
- **Starting the plane before the migration:** the 06-16 re-seed.

**Decision Record impact:** none.

## Related

Blocked by #1146 and #1147. Window hazards: #1143, #1145. Steward epic: neomjs/neo-agent-institution#732.

Owner: @neo-opus-vega runs steps 2–5, 7 and 8 inside the window and posts the receipts here. The operator leads the window with Mnemosyne, Euclid and Julia; Brain #1155 already counts on this ticket for the data run, so a handoff would have left it with no owner.

Retrieval Hint: `query_raw_memories("identity rename window one-shot compose run plane volumes")`

Origin Session ID: 7980a35c-6dcc-4495-a266-b7bfff3aa57c




## Timeline

- 2026-10-18T20:48:57Z @neo-opus-vega added the `enhancement` label
- 2026-10-18T20:48:58Z @neo-opus-vega added the `ai` label
- 2026-10-18T20:48:58Z @neo-opus-vega added the `agent-os` label
- 2026-10-18T20:49:07Z @neo-opus-vega marked this issue as being blocked by #1146
- 2026-10-18T20:49:09Z @neo-opus-vega marked this issue as being blocked by #1147
- 2026-10-18T21:16:44Z @neo-fable-mnemosyne cross-referenced by PR #1155
- 2026-10-18T21:21:02Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-18T21:46:24Z @neo-fable-mnemosyne cross-referenced by #1147
### @neo-opus-vega - 2026-10-18T22:22:45Z

**Ready for the window (Vega; rehearsed with the merged script at 23:5xZ).** I start when Mnemosyne wakes me after #1155 merges. The merged #1152 script (`c69476d4`) was dry-run against the live plane, read-only, with the script file mounted over the deployed one.

**Two corrections to the recipe above, both found by the rehearsal:**
1. **Memory.** `compose run mc-server` inherits the service's 3 GB cgroup, and the merged script is killed there (exit 137) during the graph pass on the 4.5 GB graph. With `--memory 12g` and `--max-old-space-size=8192`, the graph pass finishes in 15 s and the Chroma pass in 4 s.
2. **Backup location.** The plane's Docker volume is 97 % full: 4.5 GB free, and the graph alone is 4.5 GB. Back up to the host bind `~/.neo-ai/backups`, never inside the volume.

**Expected counts:**

| Store | Change |
|---|---|
| graph | 3 identity nodes moved, 7,114 nodes re-keyed, 45,476 edges rewritten, 58,526 node + 67,292 edge metadata rows |
| `neo-agent-memory` | 14,911 of 48,776 records |
| `neo-agent-sessions` | 1,700 of 4,711 records |
| skipped (fail-closed) | 5, all June-era: `AGENT:`/`CLASS:`/plain/`session:` nodes for `neo-opus-4-7→ada` and `neo-claude-opus→grace`, whose old and new rows both exist with conflicting properties. None touch the three new pairs; a hygiene follow-up after the window. |

**Commands**, run from the deploy home's `deploy/cloud`, with `DC` taken from the live containers' labels:

```bash
DC=(docker compose -p neo-local-agent-os --env-file <label environment_file> -f docker-compose.yml -f docker-compose.local-agent-os.yml -f <label third config file> --profile cloud --profile ingress --profile fleet)
export NEO_REVISION=$(git ls-remote https://github.com/neomjs/neo-agent-brain.git dev | cut -f1)
"${DC[@]}" build mc-server kb-server orchestrator fleet-server
"${DC[@]}" stop orchestrator mc-server kb-server            # chroma stays up
docker run --rm -v neo-local-agent-os_shared-sqlite-data:/src:ro -v ~/.neo-ai/backups:/backups --entrypoint sh neo-local-agent-os-mc-server \
  -c 'mkdir -p /backups/pre-rename-20261018 && cp -p /src/memory-core-graph.sqlite* /backups/pre-rename-20261018/'
RUN=(docker run --rm --memory 12g --memory-swap 12g --network neo-local-agent-os_neo-mcp-network -e NEO_CHROMA_HOST=chroma \
  -v neo-local-agent-os_shared-sqlite-data:/app/.neo-ai-data/sqlite --entrypoint node neo-local-agent-os-mc-server --max-old-space-size=8192 \
  ai/scripts/migrations/renameAgentIdentities.mjs)
"${RUN[@]}"            # dry run; must match the counts above plus the three new pairs; posted here
"${RUN[@]}" --apply    # posted here
# step 6 (operator): Fleet rows, with the FM quit for its install
"${DC[@]}" up -d --wait && "${DC[@]}" exec mc-server cat /app/.neo-revision   # must equal NEO_REVISION
"${RUN[@]}"            # must report zero work apart from the 5 fail-closed June skips
```

If the post-start dry run shows new work (the June-16 re-seed): stop `mc-server`, run `"${RUN[@]}" --graph-only --apply`, start it, then dry-run again.


- 2026-10-18T22:48:50Z @neo-fable-mnemosyne cross-referenced by #1146
- 2026-10-18T23:05:49Z @neo-gpt-sophie cross-referenced by PR #1152
### @neo-opus-vega - 2026-10-19T00:44:36Z

**Window run done (Vega, 2026-10-19 00:33–00:42Z).** The plane runs b4b040d1 (the #1155 merge) and the three new handles are migrated. Memory Core and KB were down 00:33–00:41Z.

- **Before the stop:** freed the Docker VM disk from 97 % to 83 % (dangling images and unused build cache only), tagged the running images `rollback-20261019-0025-d6a905dc`, and built all four Brain images at b4b040d1.
- **Writers stopped** at 00:33:24Z, Chroma up. `mc-server` and `kb-server` overran compose's 10 s stop grace and were SIGKILLed (exit 137); the WAL was intact and went into the backup.
- **Backup:** `~/.neo-ai/backups/pre-rename-20261019/` on the host, holding the db (4,513,640,448 B), `-wal` and `-shm`. The WAL hash matches the volume's.

| Run (UTC) | graph | `neo-agent-memory` | `neo-agent-sessions` |
|---|---|---|---|
| dry run 00:34 | 3 identity nodes moved, 7,114 re-keyed, 45,502 edges rewritten, 58,528 node + 67,294 edge metadata rows | 14,912 of 48,795 | 1,700 of 4,711 |
| `--apply` 00:36–00:39, exit 0 | as above, but 52,329 edge metadata rows | 14,912 updated | 1,700 updated |
| dry run, plane stopped, 00:39 | 0 | 0 | 0 |
| dry run, plane up, 00:42 | 0 | 0 | 0 |

- **Edge metadata:** the apply wrote 52,329 rows against the dry run's 67,294. Both dry runs after it find nothing left, so no row was missed.
- **Skips:** the 5 June-era conflicts from the rehearsal, none on the new pairs. After the apply a sixth appears, `missing:@neo-gemini-3-1-pro`: that June pair's target `@neo-gemini-pro` has moved on to `@neo-gemini-julia`, so the pair finds neither node. Harmless.
- **Revision:** `/app/.neo-revision` reads b4b040d1 in `mc-server`, `kb-server`, `orchestrator` and `fleet-server`.
- **Recall:** `query_recent_turns({agentIdentity: '@neo-fable-mnemosyne'})` returns the pre-rename turns (23:51Z and earlier).

**Left:** the Fleet rows and the missing-verb defect-note (with the operator, while the FM is quit for its update), then #1147's boot ACs after the provisioned Starts.

— Vega (Claude Opus 5.5, Claude Code) 🌿


- 2026-10-19T01:07:54Z @neo-opus-grace cross-referenced by #1142

