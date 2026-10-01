---
id: 351
title: An outside operator's first run provisions their own institution through the setup wizard
state: OPEN
labels:
  - agent-os
  - ai
  - epic
assignees:
  - neo-fable-clio
createdAt: '2026-09-30T13:19:28Z'
updatedAt: '2026-10-01T11:40:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/351'
author: neo-fable-clio
commentsCount: 0
parentIssue: null
subIssues:
  - '[ ] 678 ADR 0041: the bootstrap record and the verified-plane handoff'
  - '[ ] 679 First-run recipe: live step evaluation and one host-owned record'
subIssuesCompleted: 0
subIssuesTotal: 2
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
milestone: FM v1
---
# An outside operator's first run provisions their own institution through the setup wizard

Terminal predicate: on a machine that is not ours, an outside operator's cold first run of the installed Fleet Manager reaches — through the setup wizard alone, with nothing of ours in the loop — a working institution that answers a query and holds its first persisted memory, on the placement, preset and credentials they chose.

`[GRADUATED_FROM: D#18965]` — §6.2 quorum reached 2026-10-01: `claude` `[AUTHOR_SIGNAL]` (the author) + `gpt` `[GRADUATION_APPROVED]` by @neo-gpt (`DC_kwDODSospM4BHUr8`, at the Discussion body of 2026-10-01T11:01:27Z, after his OQ3 deferral was folded). The pre-quorum reservation of 2026-09-30 is promoted; leaves may be filed under this epic (ticket-create §1d satisfied). The REQUIRED Decision Record — ADR 0041, the host-record → verified-plane handoff from OQ1 — is filed beside the first record-writing leaf and gates that leaf's merge. By the operator's order of 2026-10-01 the Fleet-launched Claude Desktop seat path (neomjs/neo-agent-brain#669, #378/#379, the repackage, Ada's move) ranks above this epic's builds.

## Problem scope

The Institution's release gate ([`ROADMAP.md`](https://github.com/neomjs/neo-agent-institution/pull/336) row 1, milestone [FM v1](https://github.com/neomjs/neo-agent-institution/milestone/1)) is an outside operator, and today no outside operator can reach a plane:

- **The Fleet Manager is a projection of an Agent OS plane.** Every pane reads through the fleet server; activity, mailbox, memories and the Observatory's scene are Memory Core projections; the Knowledge Base is the one genuinely optional service; inference is hosted or local per configuration. Without a plane the shell boots honestly and shows cold states ([#12](https://github.com/neomjs/neo-agent-institution/issues/12)) — and its first screen's one action is *Connect a plane* ([#244](https://github.com/neomjs/neo-agent-institution/issues/244)), which a stranger cannot use: **no Agent OS runs in a cloud we operate** (the operator's resolution of D#18965's fork, `DC_kwDODSospM4BHQbY`, 2026-09-30). So the first run must provision.
- **Provisioning is folklore today.** The Brain's written route ([SharedDeployment](https://github.com/neomjs/neo-agent-brain/blob/dev/learn/agentos/SharedDeployment.md) and its siblings) spans 74 distinct `NEO_*` names; the Day-0 tutorial asks the operator to write an MCP client ([neomjs/neo-agent-brain#86](https://github.com/neomjs/neo-agent-brain/issues/86)); our own plane's ≈ 31 GB footprint reads as a hardware floor although it is a configuration (measured 2026-09-19: local inference 20.3 GB, Chroma 7.7 GiB at 4096 dims, services < 3 GiB; the same stack idles at ≈ 0.3 GiB).
- **Why an epic:** the outcome crosses the Brain (the recipe, the CLI bootstrap's host effects, presets over declared `AiConfig` leaves, a secret-file adapter), the Institution (the cockpit renderer inside the vessel, Home's doors) and the guides — several one-PR leaves in two repositories, one shared terminal predicate. Sized to the operator's rule: around 25 leaves, never more; a second outcome gets a successor epic.

## Intended solution shape

D#18965's convergence, in one paragraph each:

1. **One shared recipe, evaluated live.** Step definitions exist once and every renderer reads them; a step's status is a fresh observation for the bound target and the evaluated recipe version, never a remembered "completed" bit. Only what cannot be reconstructed persists — intent, consent and the receipts of host effects — in one secret-free host record owned by the bootstrap side; authenticated plane observations take over as the authority for *current* readiness once the served plane identity matches the run's target (OQ1, `[RESOLVED_TO_AC]`). A receipt is history, never health.
2. **Two renderers over one host-effect module.** The CLI bootstrap has host-effect authority and can serve the cockpit before a Brain exists; the cockpit inside the packaged vessel ([#7](https://github.com/neomjs/neo-agent-institution/issues/7), installed since 2026-09-26) is the wizard. The Discussion rejected an in-cockpit-only path because a browser page cannot run compose, probe ports or write config — the shell's main process can, so the host-effect half is one module both the vessel and the CLI call, never two implementations.
3. **Three placements, asked separately** — where the plane runs, where harnesses and workspaces run, where inference runs — each probed on the machine that bears it (RAM as a budget: total minus what the OS, the harnesses, the Docker VM cap and resident models hold; disk, cores, GPU or unified memory). This machine is the default; a cloud deployment is a placement the wizard prepares (compose, env, secret files on the operator's target), never a service of ours. **One plane per host by default** (operator, 2026-10-01: two Agent OS instances do not fit beside each other in RAM): the probe looks for a running plane first — the canonical compose project, its ingress and fleet ports — and offers Connect before Provision; a second plane is an explicit *advanced* choice sized against the Docker VM's remaining cap, which is the first ceiling, not host RAM. Measured on this machine, 2026-10-01: a 31.3 GiB Docker VM; the live Chroma resident at 11.6 GiB of a 16 GiB container cap over 15 GB on disk at 4096 dims; ≈ 20 GB of local models in LM Studio. A fixture plane with a 1024-dim embedder that shares the host's loaded models idles at 0.39 GiB (six services, 39 MB of volumes, healthy in 17 s; 2026-09-23). Wizard leaves verify against such a fixture plane, one at a time, never against a second model stack.
4. **Curated presets over declared leaves** — *hosted inference* (smallest footprint), *local small*, *local full* — each a set of env values over ADR 0019 §10.7's declared profiles, each declaring `authorityProfile`; the smallest useful stack is the fleet server, the Memory Core with its vector store and the orchestrator with hosted inference, the Knowledge Base an add-on step. Model overrides and the rest sit behind an *advanced* fold.
5. **Credentials.** The plane's own login (a GitHub or GitLab PAT, `auth.mode` `github-pat` / `gitlab-pat`), a provider key only in the hosted preset through the sanctioned `*File` sibling-leaf adapter (custody = a Compose secret + a `_FILE` env value; a named operator credential step until the adapter exists). Harness logins never enter the recipe — the operator signs in inside each harness the Fleet Manager starts.
6. **Validate before durable ingest; done is the adopter's bar.** A provider call and one observed embedding confirm the configuration, dimension included, before any corpus is ingested; the wizard witnesses its own completion — a working stack that answers a query (neomjs/neo-agent-brain#86) and first persistence (J3 in [neomjs/neo#14781](https://github.com/neomjs/neo/issues/14781)). Starting the first agent ([#171](https://github.com/neomjs/neo-agent-institution/issues/171), shipped) is a waypoint the journey consumes.
7. **Density is a measured target, not a hope.** The supported path asks three things — placement, preset, the PAT — and defaults everything else; decisions and manual actions per path are counted (the STEP_BACK's AC), checks sit behind progress and remedies, #12's inline, dismissible, resumable rule holds — no wizard wall, the frame operable underneath.
8. **Connect stays the second door** — a team member whose plane exists attaches with their own PAT (G's attach falsifier ran and held on 2026-09-23); Home ([#244](https://github.com/neomjs/neo-agent-institution/issues/244)) offers both doors.

Leaves are filed via `ticket-create`, each a one-PR deliverable with its own ACs and Contract Ledger, and linked here; the relationship graph is the registry, this body is not. Brain-side leaves live in neomjs/neo-agent-brain and link back cross-repository.

`Decision Record: REQUIRED` — narrowly, the host-record → verified-plane authority handoff and the record's ownership (OQ1 minted the boundary; ADR 0019 and neomjs/neo-agent-brain#83 do not specify it). The ADR is filed beside the first leaf that writes the record and gates that leaf's merge.

## Signal Ledger

Anchor: D#18965 body `updatedAt` 2026-09-30T13:05:51Z (`[GRADUATION_PROPOSED]`).

- `claude`: `[AUTHOR_SIGNAL by @neo-fable-clio @ body 2026-09-30T13:05:51Z]` — family coverage, not independent endorsement.
- `gpt`: *pending* — the non-author family; @neo-gpt-emmy carried both peer cycles, @neo-gpt answered OQ1; either seat's `[GRADUATION_APPROVED]` completes the quorum.
- `unknown` (@neo-preview): *pending* — welcome, not required.
- `gemini`, `kimi`: `operator_benched` — see Unresolved Liveness.

## Unresolved Dissent

None at the anchor. (The poll is open; a `[GRADUATION_DEFERRED — reason]` reopens divergence for that delta before promotion.)

## Unresolved Liveness

- `gemini`, `kimi`: `operator_benched` (2026-09), no signal expected; reactivationTrigger = the operator re-seats the family; STATUS: peer-owned liveness disposition, archived at promotion.
- `unknown` (@neo-preview): no signal yet at the anchor; STATUS: pending-peer-repoll, not required for quorum.

Tier-2 revalidationTrigger: the first cold wizard run on a non-maintainer host re-validates OQ3's thresholds and OQ8's floor (the Discussion's *Deferred dispositions* table).

## Discussion Criteria Mapping

| D#18965 criterion | Carried by |
|---|---|
| Concept §1 — create is the front door, connect the second | the wizard's door in Home and the connect path's ACs |
| Concept §2 + OQ1 — evaluated steps, the host record, the authority handoff | the recipe leaf and the record leaf; the REQUIRED ADR |
| Concept §3 + OQ3 — three placements, the budget probe, thresholds measured on the first outside host | the probe leaf; thresholds as its ACs |
| Concept §4 + OQ4/OQ8 — presets over declared leaves, the pinned dimension, the measured quality floor with its instrument | the presets leaf |
| Concept §4 + OQ5 — the plane's PAT, the `*File` secret adapter, the named credential step | the credential-step leaf (Brain) |
| Concept §5 — validate before durable ingest | the validation step's ACs in the recipe leaf |
| Concept §6 — done = brain#86's bar + J3 first persistence | the completion witness, the epic's terminal predicate |
| OQ2 — #12's inline, dismissible, resumable rule; the witness dismisses before any step ran and the frame stays operable | the cockpit renderer leaf |
| OQ6 — a cloud placement is prepared, its effects run where the plane runs | the placement leaf |
| OQ7 — neighbours keep their ownership (#14230 the contributor path, #14781 J3, brain#86 the guides, #12 the cockpit rule) | every leaf's `## Related` |
| OQ9 — the vessel bundles the pinned engine | Institution `ROADMAP.md` row 1 |
| STEP_BACK partials — recipe version mismatch shows and infers nothing; target binding (A's evidence turns no B step green); density counted; leaf writers named; receipts invalidated on target or config change; existing primitives reused (`initServerConfigs.mjs`, declared-leaf validation, the read-only deployment projection) | ACs on the leaves they name |

## Out of scope

- Automatic updates of the installed vessel (#7's signed feed) — v1 ships the documented manual update.
- The contributor path and solo mode ([neomjs/neo#14230](https://github.com/neomjs/neo/issues/14230)); the guides themselves (neomjs/neo-agent-brain#86 owns them, written against the path once it exists).
- Connect as a gated journey of its own (the ROADMAP's deferred set); an operated cloud service of ours (Option E).
- Cross-seat Neural Link mutation during setup (D#17710 joins only if a leaf needs it).

## Avoided traps / rejected shapes

- Guides only (A): rewriting cannot fix a 74-name surface. In-cockpit only (C): no host authority in a browser page — it returns only as the vessel's renderer over the shared module. Hosted-first (E): needs a service we do not operate. A generated questionnaire from leaf metadata (F): kept open as the presets' generalization, falsified if cross-leaf rules make a second rule engine.
- A stored "completed" bit (H's constraint): the cockpit showed `● streaming` over a three-week-old row. Deriving the dimension from the chosen model: withdrawn — a provider call and one observed embedding confirm it. A machine writer of `config.mjs` (OQ5): falsified — the path writes env values and secret files only.
- A remote plane with local execution: a topology falsifier, not a supported path. Attaching must never rewrite the attached plane's provider settings or data root (G's falsifier, run 2026-09-23, did not fire).

## Sweeps

(i) KB/artifact: D#18965's adjacency sweep (2026-09-19) and today's re-read of #12, #7, #244, #214, brain#86, brain#83, neo#14230, neo#14781. (ii) live latest-open queues, Institution (#349 #347 #341 #337 #335 #312 #287 #245) and Brain (#634 #632 #621 #613 #612 #609 #599 #584), read 2026-09-30 13:1xZ — Sophie-boot findings and leaves, none this outcome. (iii) Memory Core rationale sweep (`query_raw_memories`, the problem's nouns): the operator's 2026-09-19 wizard input (cloud/server vs local first, hardware check, an easy default), my measurement that ≈ 31 GB is our configuration, "CLI and wizard are two halves of one path over one ledger, #171 its last step", and the 2026-06-12 anti-50-subs rule — all folded above. (iv) own assignments: #335 here; brain#37/#50/#51/#53 — none this outcome. (v) epic layer: every open `label:epic` terminal-predicate line in the Institution, the Brain and the engine read 2026-09-30 — no overlap; neighbours #7 (the vessel), brain#212/#213 (executable profiles — the presets' substrate), brain#571 (seat folder layout — the harness placement), neo#14230 (the contributor path), neo#13015 (FM MVP: define · start · observe). Structure map: `npm run ai:structure-map -- --files --loc` run in the Brain checkout (1ac9492) — the Brain leaves join `ai/services/fleet/` and the existing host-side bootstrap scripts; the cockpit renderer joins `apps/agentos/view/`.

## Related

D#18965 · [`ROADMAP.md` row 1](https://github.com/neomjs/neo-agent-institution/pull/336) · [FM v1 milestone](https://github.com/neomjs/neo-agent-institution/milestone/1) · #12 · #7 · #244 · #214 · #171 · neomjs/neo-agent-brain#86 · neomjs/neo-agent-brain#83 · neomjs/neo-agent-brain#212 · neomjs/neo-agent-brain#213 · neomjs/neo-agent-brain#571 · neomjs/neo#14781 · neomjs/neo#14230 · ADR 0019 (§§10.3, 10.7, 10.8) · D#17710

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 4a2cca3d-9951-4e9a-b577-2a3374a22045



## Timeline

- 2026-09-30T13:19:28Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-30T13:19:30Z @neo-fable-clio added the `agent-os` label
- 2026-09-30T13:19:30Z @neo-fable-clio added the `ai` label
- 2026-09-30T13:19:30Z @neo-fable-clio added the `epic` label
- 2026-09-30T13:20:00Z @neo-fable-clio added this to the **FM v1** milestone
- 2026-09-30T13:36:52Z @neo-fable-clio cross-referenced by #19330
- 2026-09-30T13:36:58Z @neo-fable-clio cross-referenced by #335
- 2026-09-30T13:38:53Z @neo-fable-clio cross-referenced by PR #19331
- 2026-09-30T14:29:33Z @neo-gpt-emmy cross-referenced by #245
- 2026-09-30T14:32:41Z @neo-opus-vega cross-referenced by #358
- 2026-09-30T15:05:55Z @neo-fable-clio cross-referenced by #361
- 2026-09-30T15:08:28Z @neo-fable-clio cross-referenced by PR #363
- 2026-09-30T16:48:17Z @neo-opus-grace cross-referenced by #100
- 2026-09-30T17:14:48Z @neo-opus-grace cross-referenced by #644
- 2026-09-30T20:58:56Z @neo-fable-clio cross-referenced by #652
- 2026-09-30T21:55:36Z @neo-fable-clio cross-referenced by #19339
- 2026-09-30T22:00:21Z @neo-fable-clio cross-referenced by #374
- 2026-09-30T22:14:00Z @neo-fable-clio cross-referenced by #656
- 2026-10-01T10:27:17Z @neo-opus-grace cross-referenced by #571
- 2026-10-01T11:04:36Z @neo-opus-grace cross-referenced by #378
- 2026-10-01T13:03:53Z @neo-fable-clio cross-referenced by #678
- 2026-10-01T13:04:31Z @neo-fable-clio cross-referenced by #679
- 2026-10-01T13:05:22Z @neo-fable-clio added sub-issue #678
- 2026-10-01T13:05:23Z @neo-fable-clio added sub-issue #679

