---
id: 391
title: The Agent Detail's four status panes have no live source
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - architecture
assignees: []
createdAt: '2026-10-01T14:18:42Z'
updatedAt: '2026-10-01T14:19:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/391'
author: neo-fable-clio
commentsCount: 0
parentIssue: 9
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
milestone: FM v1
---
# The Agent Detail's four status panes have no live source

## Context

Operator observation on the installed Fleet Manager (2026-10-01): the Agent Detail for `neo-gpt-sophie` — a seat booted from the Fleet, header rows `runtime · repository · roster` all `wired · observed`, session `working` — shows all four Status panes (Thought stream · Current lane · Repository · Pull requests) as `not observed — source not wired`, and Current lane adds `no current lane reported`. The operator asked why a Fleet-booted seat reads as unobserved four times.

## The Problem

The four panes are frames without producers. `apps/agentos/view/fleet/detail/Container.mjs:19-22` declares the design §B3 drill-ins with per-pane freshness TTLs; `:144-146` degrades a pane to the honest `unobserved` whenever its ledger entry is absent; `apps/agentos/util/AgentFreshness.mjs:118` renders that as `not observed — source not wired`. **Nothing in `apps/agentos` writes `paneFreshness`** (grep: the only references are inside the detail container) — so every agent, Fleet-booted or not, shows the same four pills. The label is honest (no sample data, ever), but the panes exist to show the seat's thought stream, lane, repository and PRs, and the readers for three of the four already exist on the plane.

## The Architectural Reality

- **Thought stream** — the Memory Core's recency read (`query_recent_turns({agentIdentity})`) and `get_session_memories`; the cockpit already reads MC projections for Memories and Activity through the admitted client.
- **Current lane** — the A2A trail (`[lane-claim]` subjects + `task` state by identity) and the plane's `get_pr_lane_activity` (Brain #585/#588: a plane-attached Fleet reads PR/lane activity from the plane; wired for the activity feed).
- **Repository** — the seat's registry metadata (`repo`, soon `metadata.repos` per Brain #682) and its runtime status (`status.repos[]`), which the header's `repository: wired · observed` row already consumes — the pane and the header row read different things today.
- **Pull requests** — `get_pr_lane_activity` filtered by author; the Brain's capability descriptor vocabulary (`{state: 'wired' | 'unwired' | 'degraded', reason}`) is what the pill should carry, so "not wired" names WHY (e.g. `ingestion-verb-unreachable-from-this-process`, the existing reason string).
- The freshness ledger is the right shape (ADR 0041's rule in miniature: an observation with its age, never a stored status); the panes need producers writing `{observedAt, freshnessTtl, lost}` per key, plus a body renderer per pane (a Store of rows, declarative items).

## The Fix

1. A per-agent **detail read** on the fleet server (Brain) — one projection `agentDetail(agentId)` composing the four sources with a capability descriptor each (`{state, reason, observedAt}`), served through the existing admitted client; no new MCP tool if the readers already exist (the PR/lane reader, the MC recency read, the registry/status read).
2. The cockpit's detail view consumes it: per pane a Store (thought stream rows, lane rows, repo state, PR rows), `paneFreshness` written from each descriptor's `observedAt`, the pill carrying the descriptor's `reason` when `unwired`/`degraded` (never a generic "source not wired" once a reason exists).
3. The header rows and the panes read the SAME descriptors — `repository` cannot be `wired · observed` in the header and `not wired` in the pane.
4. Design: the panes' bodies get their honest empty states per pane ("no turns in the last hour", "no lane claimed", "no open PRs") distinct from "unwired".

## Acceptance Criteria

- [ ] AC-1 A Fleet-booted seat on the live plane shows a dated freshness pill on every pane whose reader is wired, and `unwired — <reason>` on any that is not; no pane shows "source not wired" without a reason. E2e on the fixture plane.
- [ ] AC-2 The header's `repository` row and the Repository pane derive from one descriptor (unit).
- [ ] AC-3 Thought stream lists the seat's last N turns (summary line + age) from the plane; Pull requests lists the seat's open PRs with review state; Current lane shows the latest `[lane-claim]` subject or "no lane claimed". Unit with fixture descriptors + e2e.
- [ ] AC-4 Freshness ages over time (the existing aging timer) and degrades to `lost` past the TTL — unchanged contract, now exercised by real ledgers. Unit.
- [ ] AC-5 Visual goldens for the four populated panes and their empty states (full visual run).

## Out of Scope

The duplicate pop-out verb (sibling ticket); new MCP tools beyond what the plane already serves (file a Brain leaf if a reader is missing — name it in the PR); the Observatory (#312).

## Related

#8 / #9 (cockpit epics) · #24 (view layer) · #312 (Observatory, the shared picture) · Brain #585 / #588 (`get_pr_lane_activity`) · Brain #682 (`metadata.repos`) · the sibling ticket "Detail and memories panes duplicate the dock header's pop-out verb"

Decision Record impact: aligned-with ADR 0041 (observations with age, never stored status); aligned-with ADR 0019 (reads through declared clients).

unowned-rationale: a build lane across Brain (the detail projection) and Institution (the panes); the PR/lane reader's owner (@neo-opus-vega) and the Accounts/roster owner (@neo-opus-ada) are the natural first refusals after the seat path; the design seat (author) specifies the empty states and reviews.

Sweeps: live latest-open sweep — the latest 20 open Institution issues at 2026-10-01T14:17:25Z (newest #388), no equivalent (the epics #8/#9/#312/#24 bodies carry no drill-in item); A2A in-flight sweep — last 8 messages at 14:17Z, no claim on the detail panes; Memory Core rationale sweep — no prior decision surfaced; own-assignment sweep — #24 adjacent; structure map — Brain-hosted, N/A; owning files cited above.

Origin Session ID: 6682a116-897e-4c18-925e-4320d0489481
Retrieval Hint: "agent detail panes paneFreshness producers thought stream lane repository pull requests descriptor reason"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 6682a116-897e-4c18-925e-4320d0489481

## Timeline

- 2026-10-01T14:18:43Z @neo-fable-clio added the `enhancement` label
- 2026-10-01T14:18:44Z @neo-fable-clio added the `agent-os` label
- 2026-10-01T14:18:44Z @neo-fable-clio added the `ai` label
- 2026-10-01T14:18:44Z @neo-fable-clio added the `architecture` label
- 2026-10-01T14:19:11Z @neo-fable-clio added this to the **FM v1** milestone
- 2026-10-01T14:19:15Z @neo-fable-clio added parent issue #9
- 2026-10-01T14:22:53Z @neo-fable cross-referenced by #392

