---
id: 1674
title: The plane has no sampler for its containers over a window
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-11-13T04:51:36Z'
updatedAt: '2026-11-13T12:53:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1674'
author: neo-opus-vega
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
closedAt: '2026-11-13T12:53:52Z'
---
# The plane has no sampler for its containers over a window

## Context

Sizing the plane rests on hand reads. #1416's per-component table was taken with `docker stats`, `/proc/meminfo` and one-off scripts in a session's scratch folder. #73 asks whether about 500 MB is a bounded working set, and its fix is "run it and measure it". #614's analysis rests on samples of `memory.stat` taken by hand.

The Brain has one windowed instrument, `ai/scripts/benchmark/serving-cost-meter.mjs` (#113's first leaf). It samples the processes that own the model endpoints. Nothing samples the plane's containers.

An instrument for the containers exists outside this repository since 2026-10-27 and has run against a live plane and against a throwaway compose project. Its ten-minute dry run covered 20 of 20 ticks with 0 gaps and 0 restarts. It was built for private measurement work. #113 draws the boundary: the method and reference-hardware results are public-safe, pricing derived from them is not. The method belongs here.

## The Problem

- `docker stats` counts active page cache as used memory (#614), so a hand read overstates what a service holds.
- One read has no percentile, no coverage and no notion of a restart. The instrument's first minute on a live plane showed CPU bursts with a p95 between 0.65 and 0.89 cores in three services; one instant shows none of that. A counter that reset reads as negative or doubled use.
- Each investigation writes its own script, so two reads cannot be compared.

## The Architectural Reality

- `ai/scripts/benchmark/` holds the instruments. Their pure cores live in `helpers/` (`servingCostCore.mjs`, `stats.mjs`) with unit specs under `test/playwright/unit/ai/scripts/benchmark/`.
- `helpers/stats.mjs#percentile` is nearest-rank over `[0, 1]` and answers 0 for an empty list.
- The meter's module doc states the honesty contract: every figure carries its coverage.
- A compose project's containers carry the labels `com.docker.compose.project` and `com.docker.compose.service`. Inside a container, cgroup v2 exposes `memory.stat`, `cpu.stat` and `io.stat`.
- The instrument needs no AiConfig: its inputs are a compose project's name and a window.
- Design authority: n/a — a new instrument; nothing existing is called wrong.

## The Fix

`ai/scripts/benchmark/plane-sampler.mjs`, with its pure core in `helpers/planeSamplerCore.mjs`:

- Per tick, one `docker exec` per container of the project reads `memory.stat` (`anon` and `file`), `cpu.stat` (`usage_usec`), `io.stat` and the interfaces' byte counters.
- Every tick is appended to `samples.jsonl`. The summary is written as `report.json`: per container (each names its service; replicas are never summed or differenced) the samples, gaps and resets, the intervals each figure rests on, resident heap (`anon`) as p50, p95 and max in GiB, CPU in cores as p50, p95 and max from `usage_usec` deltas, and network and block byte totals; the project's volume sizes at start and end; the requested window and the actual end; ticks against expected ticks; the command and a hardware slug.
- A page under `learn/agentos/measurements/` names the method and what each figure leaves out.

## Contract Ledger

| Target surface | Source of authority | Proposed behaviour | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `node ai/scripts/benchmark/plane-sampler.mjs --project <name> --hardware <slug> --interval <duration> --window <duration> --out <dir>`, new | this ticket; the meter's honesty contract | samples until the window ends, writes `samples.jsonl` and `report.json`, prints both paths | no running container in the project, or a missing required flag: exits non-zero and says which. SIGINT or SIGTERM inside the window: the report is written with `stoppedEarly: true` and the actual end | module JSDoc; the measurements page | a run against a live plane, in the pull request |
| `report.json`, new | this ticket | the fields under The Fix | a failed read counts as a gap and adds no sample; a failed or malformed I/O or network part is `null`, an empty one is 0; an interval in which any counter went backwards is dropped whole and counted as a reset; a figure without a usable interval is `null`; expected ticks round up (a tick at the start and one per full interval inside the window); a volume belongs by the project's label | the measurements page | unit specs of the core |

Decision Record impact: none.

## Acceptance Criteria

- [ ] The core has unit specs for: a cgroup read, a duration, docker's size strings, a restart inside a series (its delta dropped and counted), a gap, and a service without samples (`null`, never 0).
- [ ] One percentile definition: the core uses `helpers/stats.mjs#percentile`.
- [ ] Resident heap is `memory.stat`'s `anon`. The file-backed bytes stay beside it in `samples.jsonl` and are not added to it.
- [ ] A stop inside the window writes a report that says so; a stop after the window is not an early stop.
- [ ] The pull request holds a run of at least ten minutes against a live plane: its ticks equal the expected ticks, and its heap figures stand beside one `docker stats` read of the same minute.
- [ ] The measurements page says what the figures leave out: the VM's own overhead, page cache, and everything outside the compose project.
- [ ] The script reads no AiConfig and writes nothing outside `--out`.

## Out of Scope

- Host processes (seats, bridges, the model server): #1416's table and the serving-cost meter.
- A daemon or health producer that samples all the time. This instrument runs on demand.
- METRIC nodes: #113 owns that discipline for cost figures.
- Any figure derived for pricing.

## Avoided Traps

- **`docker stats`.** It counts active page cache as used memory (#614).
- **Filling a missing tick.** A gap is reported as a gap, never interpolated.
- **Reading cgroups from the host.** On macOS the containers live in a VM. `docker exec` reaches the same files on every host the plane runs on.

## Related

#113 (the measurement program; this is a sibling of its meter), #1416, #73, #614.

Ownership: self-assigned. The existing implementation is in my hands and is not public, so this is not a contributor ticket.

Live latest-open + closed sweep: latest 20 of 159 open (red band; #113 is the owning program and has no leaf for the containers) and searches over all states ("container sampler", "cgroup memory", "docker stats page cache", "plane sizing instrument", "serving-cost-meter"; Out of Scope and Avoided Traps of #614 read) at 2026-11-13T04:50Z; no equivalent and no exclusion. A2A in-flight claim sweep: the latest 12 messages at 04:50Z and the 30 before them at 04:10Z, all read states; no claim on a plane sampler. MC sweep: "plane container memory read by hand, docker stats counts page cache, no repeatable per-container sampler for sizing the plane", 5 results, no prior decision. Own-assignment sweep: 15 open in this repository; #1416's body read, #1434 and #1432 measure other things by their titles (dead index elements, a heal operation). Structure-map gate: `ai/scripts/benchmark/` with `helpers/`; siblings `serving-cost-meter.mjs` and `helpers/servingCostCore.mjs`.

Origin Session ID: ea0a0782-b7f5-46ef-92a6-7e86e889ac6f

Retrieval Hint: "plane container sampler cgroup anon heap cpu cores window percentile coverage restart gap"

— Vega (Claude Fable 5.2, Claude Code) 🌿

## Timeline

- 2026-11-13T04:51:36Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-13T04:51:37Z @neo-opus-vega added the `enhancement` label
- 2026-11-13T04:51:37Z @neo-opus-vega added the `ai` label
- 2026-11-13T05:13:52Z @neo-opus-vega referenced in commit `8def4bc` - "feat(benchmark): the plane sampler's pure core (#1674)

Parses one container's cgroup v2 read, folds ticks into per-service series and builds the window
report. Heap is the anonymous memory alone, a failed read is a gap, a backwards counter is a
restart whose delta is dropped, and a service without samples reports null."
- 2026-11-13T05:13:52Z @neo-opus-vega referenced in commit `a412cef` - "feat(benchmark): the plane sampler's CLI and its measurements page (#1674)

plane-sampler.mjs samples one compose project's containers over a window through docker exec and
writes samples.jsonl and report.json into --out. The page records the method, what the figures
leave out, and one reference run beside a docker stats read of the same second."
- 2026-11-13T05:13:54Z @neo-opus-vega cross-referenced by PR #1675
- 2026-11-13T05:24:09Z @neo-opus-vega referenced in commit `5721e94` - "chore(benchmark): the sampler's byte sizes are products, not exponents (#1674)

lint-retry-bounds classifies every exponent expression under ai/. Two byte sizes written as
2 ** n were unclassified candidates and turned the unit job red. Written as products they are no
growth expressions, and the registry needs no entry for them."
- 2026-11-13T06:03:46Z @neo-opus-vega referenced in commit `02ca7f4` - "fix(benchmark): the sampler keeps one series per container and tells unknown from zero (#1674)

A series is one container's: replicas of a service are never differenced or summed. An interval in
which any counter went backwards is dropped whole, a failed or malformed I/O or network part is
null while an empty one is 0, and a figure without a usable interval is null. Volumes belong by the
project's label. A window counts a tick at its start and one per full interval, ends with the
window, and a stop ends its wait at once. The page's reference run is retaken at this code."
- 2026-11-13T06:33:39Z @neo-opus-vega referenced in commit `4c80f23` - "test(benchmark): the sampler's CLI arms run one at a time (#1674)

Each arm starts the CLI and its stand-in docker as real processes. The unit suite runs a file's
tests in parallel, so the three arms started together in three of the four workers."
### @neo-opus-vega - 2026-11-13T06:41:18Z

**Handover, 2026-11-13 06:41Z** (session sunset at context exhaustion; the lane stays mine).

- PR #1675 is at `4c80f237`, green (15 pass, 2 skipped). Emmy's Round 1 (three Required Actions) is answered in its comment 6845552790, and she is re-requested. Her Round 2 is a disposition; on approval the merge handoff follows.
- The receipts of the page's reference run are in my Brain checkout under `.git/receipt-1674-run2/` (`report.json`, `samples.jsonl`).
- Two red `unit` runs at `02ca7f4f` were retried timing tests in files the PR does not touch; the PR body's last paragraph and the defect-note of 06:33Z have the specimens.
- After the merge, outside this repository: the private copy of the sampler is deleted and its runner points at this script.

— Vega (Fable 5.2, Claude Code) 🌿

- 2026-11-13T12:53:52Z @tobiu referenced in commit `f21dfc8` - "feat(benchmark): the plane has a sampler for its containers (#1674) (#1675)

* feat(benchmark): the plane sampler's pure core (#1674)

Parses one container's cgroup v2 read, folds ticks into per-service series and builds the window
report. Heap is the anonymous memory alone, a failed read is a gap, a backwards counter is a
restart whose delta is dropped, and a service without samples reports null.

* feat(benchmark): the plane sampler's CLI and its measurements page (#1674)

plane-sampler.mjs samples one compose project's containers over a window through docker exec and
writes samples.jsonl and report.json into --out. The page records the method, what the figures
leave out, and one reference run beside a docker stats read of the same second.

* chore(benchmark): the sampler's byte sizes are products, not exponents (#1674)

lint-retry-bounds classifies every exponent expression under ai/. Two byte sizes written as
2 ** n were unclassified candidates and turned the unit job red. Written as products they are no
growth expressions, and the registry needs no entry for them.

* fix(benchmark): the sampler keeps one series per container and tells unknown from zero (#1674)

A series is one container's: replicas of a service are never differenced or summed. An interval in
which any counter went backwards is dropped whole, a failed or malformed I/O or network part is
null while an empty one is 0, and a figure without a usable interval is null. Volumes belong by the
project's label. A window counts a tick at its start and one per full interval, ends with the
window, and a stop ends its wait at once. The page's reference run is retaken at this code.

* test(benchmark): the sampler's CLI arms run one at a time (#1674)

Each arm starts the CLI and its stand-in docker as real processes. The unit suite runs a file's
tests in parallel, so the three arms started together in three of the four workers."
- 2026-11-13T12:53:54Z @tobiu closed this issue

