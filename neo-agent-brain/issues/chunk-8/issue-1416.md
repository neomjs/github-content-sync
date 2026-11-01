---
id: 1416
title: 'Measure and reduce the Agent OS''s resident memory on a host: the plane, ten seats, their MCP bridges'
state: OPEN
labels:
  - enhancement
  - ai
  - performance
  - agent-os
assignees: []
createdAt: '2026-11-01T02:47:26Z'
updatedAt: '2026-11-01T02:47:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1416'
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
# Measure and reduce the Agent OS's resident memory on a host: the plane, ten seats, their MCP bridges

## Context

Design authority: the operator, Nov 1 03:5x CET — "worth an investigation ticket: reducing the RAM use of our very own neo agent os" — after his Activity Monitor read at 03:3x CET during one model run: 117.8 of 128 GB used, 46.2 GB app memory, 18.0 GB wired, 52.6 GB compressed, 6.5 GB swap, memory pressure in the warning band. Prior art in Memory Core: Clio, 2026-10-01 — only one Agent OS instance fits per host, the plane and seats take 31+ GB; Clio, 2026-09-23 — host tiering from a 127/128 GB read (64 GB hosts run local models with capped VMs, 32 GB hosts default to remote models, 16 GB remote-only); Vega, 2026-08-11 — a high footprint traced to allocation, not capacity (neo #16998); Ada, 2026-09-24 — a plane recreate with a 1 GiB memory reset and an OOM watcher. Sweeps at 03:5x CET: the latest 20 open Brain issues (#1415 … #1005) and the closed and open searches on "memory footprint RAM resident process plane" return no equivalent; no A2A claim on it in the last hour; the Memory Core rationale sweep returned the four records above. Filed by a Fable seat as intake only; the investigation is a non-Fable seat's.

## The Problem

The operator's machine runs the whole organism — the container plane and ten peer seats, each with its harness session and its own MCP bridge processes — and sits near full memory before any model run: the one TRELLIS.2 F16 run of tonight (7.6 GB resident plus its Metal working set) pushed it into swap. The top resident processes at that moment: the Virtualization service carrying the plane's containers at 10.9 GB, LM Studio's node server at 7.3 GB (not ours), three Codex helpers at 2.2, 0.9 and 0.8 GB, and the rest spread over ten harness sessions and their bridges — unmeasured as a set. Nobody has a per-component table of what the Agent OS itself holds resident, so nobody can say which 20 GB are ours to give back, or what a 32 GB host (the tiering target) would need removed.

## The Architectural Reality

- **The plane** runs as containers (KB with its Chroma store, Memory Core with the SQLite holder and graph, the orchestrator) under macOS virtualization — one process in Activity Monitor, its memory a VM budget rather than the services' own use (`ai/` deployment prescriptions; the plane-cut notes of Oct 31 name four services).
- **Each seat** runs a harness (Claude Code, Codex Desktop, Antigravity, Kimi) that spawns the seat's MCP servers as host processes: `neo-mjs-knowledge-base`, `neo-mjs-memory-core`, `neo-mjs-neural-link`, `neo-mjs-github-workflow` — per session, so ten seats mean up to forty bridge processes, each a Node runtime with its own heap and caches (the restart notes: NL and github-workflow are host-spawned; KB/MC/OC are containers).
- **The daemons** (wake, orchestrator, backup, REM) and the Fleet Manager app are long-lived Node processes on the host.
- The harness binaries themselves are the vendors' and out of scope; what we launch and how many of each is ours.

## The Fix (an investigation, then leaves)

1. **Measure, per component, on this host:** a table of resident, compressed and wired memory for the plane's VM and each container inside it, each daemon, the Fleet Manager, and the bridge processes grouped by service and by seat — captured at idle and during one model run; private bytes, so a reduction frees what it says (Ada's disk method applied to memory).
2. **Rank the reductions by GB saved per effort**, candidates to confirm or strike by the table: one shared bridge process per service for all seats instead of one per seat; memory limits and cache sizing on the plane's containers (the Chroma and SQLite resident sets); idle-seat hibernation (a seat with no turn in N hours releases its bridges); Node heap caps on the daemons; whatever the table surfaces that nobody guessed.
3. **Prove one** with a before/after on this host, and restate the tiering target (what a 32 GB host would need) from the measured table.

## Acceptance Criteria

- [ ] AC-1 The per-component memory table for this host exists on this ticket (idle and during one model run), private bytes, with the capture method named.
- [ ] AC-2 At least three reductions are ranked by measured GB per effort, each with the component and the mechanism; guesses struck by the table are listed as struck.
- [ ] AC-3 One reduction is implemented and measured before/after on this host, as its own leaf or PR.
- [ ] AC-4 The host-tiering target (32 GB) is restated from the table: what runs, what is removed, what moves remote.

## Out of Scope

The harness vendors' own memory; LM Studio and other operator-run models; disk space (Ada's hygiene round of Nov 1); a new scheduler or plane architecture.

## Avoided Traps

- Capacity claims without per-process measurement (Vega's Aug 11 finding: allocation, not capacity).
- Killing seats or sessions as the fix: the OS must fit beside ten seats, not instead of them.
- Measuring the VM's budget as the services' use: measure inside the containers too.

## Related

Memory Core records cited above; neo #16998; the plane-cut notes of Oct 31 (Vega); the disk hygiene round (Ada, Nov 1).

Ownership: `handoff: @neo-opus-vega` (the plane's owner through the Oct 31 cuts; Ada as alternate) — a non-Fable seat per the operator's delegation rule; no Fable tokens on the measurement.

Origin Session ID: 65ddc71f-04c4-4c7d-90f0-126e11f23681
Retrieval Hint: "agent os resident memory per component plane seats MCP bridges host tiering 32 GB"

Mnemosyne — Fable 5.1, Claude Code

## Timeline

- 2026-11-01T02:47:28Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-01T02:47:28Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-01T02:47:28Z @neo-fable-mnemosyne added the `performance` label
- 2026-11-01T02:47:28Z @neo-fable-mnemosyne added the `agent-os` label

