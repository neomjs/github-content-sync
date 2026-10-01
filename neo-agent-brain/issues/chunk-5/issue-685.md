---
id: 685
title: The wizard's placement probe reads host and guest RAM budgets apart
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-fable-clio
createdAt: '2026-10-01T13:31:24Z'
updatedAt: '2026-10-01T20:02:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/685'
author: neo-fable-clio
commentsCount: 0
parentIssue: 351
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-01T20:02:58Z'
---
# The wizard's placement probe reads host and guest RAM budgets apart

## Context

Epic neomjs/neo-agent-institution#351 (graduated from neomjs/neo#18965 on 2026-10-01), shape point 3: *three placements, asked separately* — where the plane runs, where harnesses and workspaces run, where inference runs — and a probe that reads **the machine that will bear each workload**. The Discussion's OQ3 was dispositioned `[DEFERRED_WITH_TIMELINE]` onto this leaf with its contract fixed at the 11:01:27Z fold: **two budgets kept apart**, host and guest; a configured cap is a limit, never consumption; no consumer counted twice; a backend whose host reservation cannot be observed reports that uncertainty or applies a *named* conservative policy; **one plane per host by default** — the probe looks for a running plane first and offers Connect before Provision.

Parent: neomjs/neo-agent-institution#351 (sub-issue link set after creation). Consumer: the placement step of the first-run recipe (#679); the thresholds come from the presets leaf (filed beside this one), never from this module.

## The Problem

The earlier OQ3 expression subtracted the Docker VM's cap from the host as if it were consumed. @neo-gpt's counterexample (`DC_kwDODSospM4BHUkC`): a 64 GiB host, 14 GiB other use, 20 GiB resident models, a 32 GiB VM cap at 2.5 GiB residency — the old expression read −2 GiB where host headroom is 27.5 GiB and guest headroom 29.5 GiB. The same conflation sat in my 2026-09-23 host reading. A probe that gets this wrong steers a 64 GiB machine to hosted inference it does not need, or a 32 GiB machine into a local preset that swaps.

The other failure is the maintainer host itself (2026-09-23 and 2026-10-01): 128 GiB total, `vm_stat` free 0.1 GiB with 24.6 GiB compressed and 17 GiB of swap, while `memory_pressure` reports 70 % free from cache accounting. `os.freemem()` is not availability on macOS; a probe that reads it as headroom recommends a local preset to a swapping machine.

## The Architectural Reality

Measured instruments on this host (2026-10-01, names only, no values that identify the machine beyond its class):
- Host total: `sysctl hw.memsize` / `os.totalmem()` — portable.
- Docker Desktop's VM: `docker info --format '{{.MemTotal}} {{.NCPU}} {{.OperatingSystem}}'` → the VM's total = **the cap** (Docker Desktop's documented default is 50 % of host RAM — the silent first-run cost), its CPUs, and the guest OS name (here Ubuntu 24.04 inside the VM — a tell that a VM exists). Guest **residency** = `docker stats --no-stream` per container (sum, inside the VM). The VM's **host reservation** on macOS is observable as the virtualization process's RSS (`ps`), counted ONCE as a host consumer — never the containers' sum (inside the VM) and never the cap.
- Linux without a VM: no guest budget (`guest: null`); containers are host consumers directly, their RSS counted once.
- Resident models: `lms ps` lists loaded models with sizes and `IDLE` state — an idle loaded model still holds its weights (mmapped); counted once as a host consumer. Ollama has the equivalent (`ollama ps`).
- Our own plane: idles at 0.39 GiB and peaks ≤ 2.5 GiB on the fixture (`fm-fresh-small`, 2026-09-23); the live plane's Chroma sits at 11.6 GiB because of a 4096-dim HNSW over this org's corpus — a configuration, not a floor (D#18965 Evidence). **Thresholds never come from our plane**; they are the presets leaf's data.
- A plane that already runs: the canonical compose project (`docker compose ls --format json`), its ingress and fleet ports — the one-plane-per-host rule's detector.
- The deployment-state projection and the healthcheck's served `plane.id` are the owners of plane readiness (ADR 0041 §2.3); the probe reads the MACHINE, never the plane's health.

## The Fix

1. **`probePlacement({target, readers})`** — one module beside #679's recipe/record module (same placement pre-flight; `ai/services/fleet/` is the sibling-pattern candidate, a new `ai/services/setup/` runs the full structural pre-flight in the PR). Output, one object per probed machine:
   `{host: {totalBytes, consumers: [{name, bytes, source, population}], containers, availableBytes, complete, pressure: 'ok' | 'swapping' | 'unknown'}, guest: {backend, capBytes, cores, guestOs, residencyBytes, availableBytes, reservationPolicy, complete} | null, disk: {rootFreeBytes}, cores, accelerator: {kind, memoryBytes} | null, runningPlane: {project, status, ports, configFiles} | null, observed: {<reader>: Boolean}, uncertainty: [{reader, reason}], probedAt, target}` *(shape reconciled 2026-10-01 with PR #707 on review RA-4: `uncertainty` and `observed` are top-level, keyed by reader; `runningPlane` carries the compose status and config files, not a revision — the plane's revision is the deployment-state projection's, never the probe's)*.
   - **host.availableBytes = total − Σ consumers** (OS floor, harnesses, resident models, the VM's host reservation), **each consumer once**; a cap never appears in the sum. **One owner per process population:** the host inventory comes back partitioned (the VM's own processes, each model server, everything else); the VM population is replaced by the observed reservation only when a VM topology was observed, a model server's resident set only by an inventory that actually lists its weights, and container processes on a host-native engine are already in the inventory and never added a second time.
   - **guest.availableBytes = cap − residency** where the plane will run in a VM; `guest: null` where the engine was observed running on the host itself. An unobserved topology is neither.
   - **reservationPolicy** names what was done when the VM's host reservation was unobservable (e.g. `'vm-reservation=residency+2GiB'`), or `uncertainty` carries the reason — never a silent number.
   - **complete** (host and guest): false when any reader the budget is computed from failed or answered an unusable shape; an incomplete budget has `availableBytes: null` and **never yields an affirmative fit**.
   - **pressure: 'swapping'** when the host reports swap in use or compressed memory above a named share (the one named classifier in the module; preset sizing thresholds are #686's data); the probe then refuses to call any local preset a fit, whatever the arithmetic says.
2. **`fitsPreset(probe, presetWorkload)`** — pure: compares the preset's declared workload (`{planeIdleBytes, planePeakBytes, modelsBytes, vmCapRecommendedBytes}` — the presets leaf's data) against BOTH budgets and returns `{fits, margins: {host, guest}, reasons}`. The workload is **additional demand** of a new plane: the host backs the plane's peak and its models with or without a VM (VM memory is host memory), the guest must hold the same peak under its cap; raising the cap alone never moves the host margin. With no preset data it returns `null`, never a verdict; a malformed workload or an incomplete budget is a refusal, never a zero-demand fit.
3. **Backend readers** are injected (`readers: {totalmem, cores, hostUse, vmInfo, containerStats, vmReservation (optional), loadedModels, swap, composeLs, composePorts, statfs, accelerator}`; `createDefaultReaders({run})` builds the production shells over an injectable command runner) so every arm is unit-testable with fixtures and the production readers run on fixture command output; a reader that fails reports `uncertainty` and marks its budget incomplete, never throws the probe.
4. **Remote targets**: the probe never guesses a machine it does not run on — the CLI (#679) runs it ON the target (`--probe --json`) and the cockpit shows that JSON (`target.kind: 'local' | 'remote-json'`); a cloud placement without a probe result is marked `unprobed` in the recipe's placement step.
5. **`runningPlane`** detection feeds the recipe's placement step: a canonical plane found → Connect is offered first; a second plane is the advanced choice sized against `guest.availableBytes`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `probePlacement()` output shape | D#18965 OQ3 disposition (11:01:27Z body); epic point 3 | two budgets, consumers listed once each, pressure + uncertainty explicit | a failed or unusable reader → `uncertainty` entry and the budget it feeds marked `complete: false` with `availableBytes: null`; an incomplete budget never yields an affirmative fit | module JSDoc | unit: the counterexample fixture → host 27.5 GiB / guest 29.5 GiB; a failed model inventory → no fit |
| the host budget rule | OQ3: "a cap is a limit, never consumption" | `total − Σ consumers`, each once; the VM counted by its host reservation only | unobservable reservation → named policy or uncertainty | JSDoc | unit: raising only the VM cap leaves `host.availableBytes` unchanged |
| the guest budget rule | OQ3 | `cap − residency`; `null` where the engine was observed on the host itself; an unobserved topology is neither | — | JSDoc | unit: the native-engine fixture → `guest: null`, the container stats listed under `host.containers` for display only, their processes already counted once in the host inventory |
| `pressure` | the 2026-09-23 specimen (127 GB used, 17 GB swap) | `'swapping'` refuses every local fit | `'unknown'` when swap/compression are unreadable | JSDoc | unit: the specimen fixture → no local preset fits |
| `fitsPreset()` | the presets leaf's declared workloads | pure comparison against both budgets | no preset data → `null` | JSDoc | unit: a 64 GiB fixture fits local-full, a 32 GiB fixture only hosted |
| `runningPlane` | epic point 3 (one plane per host) | canonical compose project + ports detected → Connect first | compose unreadable → `null` + uncertainty | JSDoc + `SharedDeployment.md` pointer | unit: injected `compose ls` fixture |

## Acceptance Criteria

- [ ] AC-1 Euclid's fixture (64 GiB host, 14 GiB other, 20 GiB models, 32 GiB VM cap, 2.5 GiB residency; readers injected): `host.availableBytes` = 27.5 GiB, `guest.availableBytes` = 29.5 GiB, every consumer listed once. Unit.
- [ ] AC-2 The falsifier: the same fixture with only the VM cap raised to 48 GiB → `host.availableBytes` unchanged, `guest.availableBytes` + 16 GiB. Unit.
- [ ] AC-3 The swapping-host specimen (total 128 GiB, swap in use, compressed ≥ the named share): `pressure: 'swapping'`; `fitsPreset()` returns `fits: false` for every local preset with the reason, whatever the arithmetic says; `os.freemem()` is never read. Unit.
- [ ] AC-4 Linux without a VM: `guest: null`, containers appear once in `host.consumers`. Unit.
- [ ] AC-5 Unobservable VM reservation: the output names the policy applied or carries the uncertainty; a snapshot assertion fails on any silent number. Unit.
- [ ] AC-6 `runningPlane`: an injected `compose ls` fixture with the canonical project → `runningPlane` set with ports; the recipe's placement step (#679) consumes it to offer Connect first. Unit here; the step wiring in #679.
- [ ] AC-7 Without preset data `fitsPreset()` returns `null`; with the presets leaf's table a 64 GiB fixture fits `local-full` and a 32 GiB fixture fits only `hosted`. Unit.
- [ ] AC-8 `[L4-deferred — operator handoff needed; Residual-Owner: neomjs/neo-agent-institution#351]` *(post-merge, first outside host)* the probe's JSON recorded on the epic; the density count unaffected (one question, no manual action). This observation describes the target machine for the recipe's placement step and is kept apart from the plane's readiness (ADR 0041 owners).

## Out of Scope

The preset table and its thresholds (the presets leaf); the cockpit's rendering of the probe (Institution leaf); GPU / unified-memory *sizing* beyond reporting what `system_profiler` / `nvidia-smi` expose (a follow-up when a local preset needs it); sizing a second plane beyond the guest budget; any plane-health read (ADR 0041 owners).

## Avoided Traps

Subtracting a cap as consumption (the OQ3 bug, fixed in the Discussion). Counting the VM twice (its host RSS AND its containers). `os.freemem()` / `memory_pressure` as availability on macOS. A loaded-idle model read as free memory. Our plane's 31 GB as a threshold. A probe that guesses a remote machine.

## Related

neomjs/neo-agent-institution#351 (parent) · #679 (the recipe's placement step and the CLI that runs this probe) · the presets leaf (thresholds as data) · neomjs/neo#18965 OQ3 + Concept §3 · ADR 0041 (a probe result is an observation for the bound target, never stored as status) · ADR 0019 §10.7 (plane placement is a declared leaf; the probe informs the declaration, never writes it) · `ai/daemons/orchestrator/services/ContainerHealthDiagnosisService.mjs` (the existing `docker stats` reader; a shared reader is the implementer's call)

Decision Record impact: aligned-with ADR 0041; aligned-with ADR 0019 (no `process.env` read; the probe reads the machine, not config).

unowned-rationale: a build lane behind the Claude Desktop seat path (operator priority 2026-10-01); the container-stats reader's owner (@neo-opus-vega, #593/#594) is the natural first refusal, offered by DM when the seat path clears; the design seat (author) answers on the ticket and reviews.

Sweeps: live latest-open sweep — the latest 20 open Brain issues read at 2026-10-01T13:29:31Z (newest #684), no equivalent; A2A in-flight sweep — the last 12 messages at 13:29Z, all read-states, no `[lane-claim]` / `[lane-intent]` on a placement probe (claims: #682/#683 multi-repo seat, #684 GitLab seat, #681 rebase); Memory Core rationale sweep (`query_raw_memories`) — my 2026-09-23 host measurement (Docker Desktop VM cap = 50 % default; the tier derivation) and the 2026-10-01 one-plane-per-host measurement, both folded into D#18965; @neo-opus-vega's 2026-08-13 finding that compose honours memory limits but not CPU limits (a cap and a consumption are different instruments); own-assignment sweep — #678, #37, #50, #51, #53, none on this surface; structure map (`npm run ai:structure-map -- --files --loc`, exit 0, 13:0xZ) — `ai/services/fleet` holds the plane readers (`planeDeploymentStateReader.mjs`, `projectDeploymentStateForFleet.mjs`), no `ai/services/setup`.

Origin Session ID: 6682a116-897e-4c18-925e-4320d0489481
Retrieval Hint: "placement probe two budgets host guest VM cap residency swapping host one plane per host"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 6682a116-897e-4c18-925e-4320d0489481



## Timeline

- 2026-10-01T13:31:27Z @neo-fable-clio added the `enhancement` label
- 2026-10-01T13:31:27Z @neo-fable-clio added the `ai` label
- 2026-10-01T13:31:28Z @neo-fable-clio added the `architecture` label
- 2026-10-01T13:31:28Z @neo-fable-clio added the `agent-os` label
- 2026-10-01T13:32:31Z @neo-fable-clio added parent issue #351
- 2026-10-01T13:38:29Z @neo-fable-clio cross-referenced by #384
- 2026-10-01T15:16:05Z @neo-fable-clio cross-referenced by #697
- 2026-10-01T17:12:16Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-01T17:20:07Z @neo-fable-clio cross-referenced by PR #707
- 2026-10-01T17:29:21Z @neo-fable-clio referenced in commit `62d138a` - "feat(fleet): the placement probe reads host and guest RAM budgets apart (#685)"
- 2026-10-01T18:21:27Z @neo-fable-clio cross-referenced by #713
- 2026-10-01T18:21:51Z @neo-fable-clio cross-referenced by #714
- 2026-10-01T18:23:28Z @neo-fable-clio cross-referenced by #686
- 2026-10-01T18:25:04Z @neo-fable-clio cross-referenced by PR #715
- 2026-10-01T18:30:26Z @neo-fable-clio cross-referenced by #351
- 2026-10-01T18:31:05Z @neo-fable-clio cross-referenced by #679
- 2026-10-01T19:07:28Z @neo-fable-clio referenced in commit `e8516ad` - "feat(fleet): the placement probe reads host and guest RAM budgets apart (#685)"
- 2026-10-01T19:07:28Z @neo-fable-clio referenced in commit `9c1bda2` - "feat(fleet): the probe refuses a fit on an incomplete budget, counts each process population once, and backs guest growth on the host (#685)"
- 2026-10-01T19:23:27Z @neo-fable-clio referenced in commit `8c41e84` - "feat(fleet): the placement probe reads host and guest RAM budgets apart (#685)"
- 2026-10-01T19:23:27Z @neo-fable-clio referenced in commit `ee085c1` - "feat(fleet): the probe refuses a fit on an incomplete budget, counts each process population once, and backs guest growth on the host (#685)"
- 2026-10-01T20:02:58Z @tobiu referenced in commit `78366b7` - "feat(fleet): the placement probe reads host and guest RAM budgets apart (#685) (#707)

* feat(fleet): the placement probe reads host and guest RAM budgets apart (#685)

* feat(fleet): the probe refuses a fit on an incomplete budget, counts each process population once, and backs guest growth on the host (#685)"
- 2026-10-01T20:02:59Z @tobiu closed this issue

