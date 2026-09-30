---
id: 338
title: CARD-CONTRACT.md still prescribes replacing a sample-seed count
state: CLOSED
labels:
  - bug
  - documentation
  - agent-os
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-09-30T08:29:00Z'
updatedAt: '2026-09-30T09:47:41Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/338'
author: neo-fable-clio
commentsCount: 1
parentIssue: 237
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-30T09:47:41Z'
---
# CARD-CONTRACT.md still prescribes replacing a sample-seed count

## Context

#237's terminal predicate — the cockpit carries no sample data — is delivered in the app: #238 (PR #250), #239 (PR #254) and #240 (PR #324) are closed, and Euclid's installed-product witness of 2026-09-29 ([#237 comment](https://github.com/neomjs/neo-agent-institution/issues/237#issuecomment-5893588146)) shows the rebuilt bundle with a zero-agent roster behind *Add your first agent*, real retained activity and a sample-free Tasks pane. The same handoff carries Vega's source audit: `apps/agentos/CARD-CONTRACT.md:15` still describes replacing a sample-seed lane count, "normative prose [that] should be reconciled in the owner's closeout". This leaf is that reconciliation, so the epic can close on a contract that does not prescribe a retired mechanism.

## The Problem

Line 15 (the open-lane count badge row) ends its fallback cell with: *"the first authoritative load REPLACES any sample-seed count with this live truth"*. There is no seeded count any more — the roster store starts empty (#239 AC-1) — so the sentence prescribes a step against a source that does not exist. A contract row that names a retired seed teaches the next reader the wrong model (the same drift class #237 closed in code), and `grep -rn -i sample apps/agentos --include='*.md'` finds this one line and nothing else.

## The Architectural Reality

`apps/agentos/CARD-CONTRACT.md` is the roster card's normative contract (one row per slot: source field · rendering rule · fallback). The badge row's semantics are correct and stay: `null` (not stamped) or `0` (nothing open) → no badge; unknown never poses as zero. Only the trailing clause is stale. The runtime already behaves as the corrected sentence says: `mapRosterRow` carries `openLaneCount` from the DTO, and a landing (`AgentOS.util.FleetAdmission`, #250) is the only way rows enter the store.

## The Fix

One-line edit of the fallback cell in line 15: drop "the first authoritative load REPLACES any sample-seed count with this live truth" and close the cell after "zero is not badge value" — the first load is the only load, so there is nothing to replace. No other file mentions a sample seed.

## Acceptance Criteria

- [ ] AC-1 `grep -rn -i sample apps/agentos --include='*.md'` returns nothing.
- [ ] AC-2 The badge row keeps its `null` / `0` → no-badge semantics verbatim; the diff touches only the stale clause.

## Out of Scope

The badge's producer (a Brain-side enricher stamping `openLaneCount`) and the roster DTO — unchanged. The rest of the contract's rows — unchanged.

## Related

#237 (parent — the closeout this leaf completes) · #239 · #240 · #250 · #254 · #324
Live latest-open sweep: checked the latest 20 open issues at 2026-09-30T08:29Z; no equivalent (newest: #337, #335, #312, #287, #247). A2A claim sweep: no claim on this scope. Memory Core sweep: the handoff comment above is the only prior mention. Own-assignment sweep: #237 (this leaf's parent), #335 — neither is this. Structure-map gate: n/a (a documentation line).
Decision Record impact: none.

Origin Session ID: 4a2cca3d-9951-4e9a-b577-2a3374a22045
Retrieval Hint: "CARD-CONTRACT sample-seed count clause #237 closeout"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 4a2cca3d-9951-4e9a-b577-2a3374a22045

## Timeline

- 2026-09-30T08:29:01Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-30T08:29:02Z @neo-fable-clio added the `bug` label
- 2026-09-30T08:29:02Z @neo-fable-clio added the `documentation` label
- 2026-09-30T08:29:02Z @neo-fable-clio added the `agent-os` label
- 2026-09-30T08:29:03Z @neo-fable-clio added the `ai` label
- 2026-09-30T08:29:34Z @neo-fable-clio added parent issue #237
- 2026-09-30T08:30:07Z @neo-fable-clio cross-referenced by #237
- 2026-09-30T08:30:54Z @neo-fable-clio cross-referenced by PR #339
- 2026-09-30T08:34:21Z @neo-fable-clio referenced in commit `b286b7a` - "test(visual): re-stamp the baseline inputs after the contract edit (#338)"
- 2026-09-30T09:02:18Z @neo-fable-clio cross-referenced by #340
- 2026-09-30T09:45:49Z @tobiu referenced in commit `2557715` - "Merge pull request #339 from neomjs/clio/338-card-contract-seed

docs(agentos): the card contract names no sample-seed count (#338)"
### @neo-fable-clio - 2026-09-30T09:47:40Z

Landed via #339 (merged 2026-09-30 09:45:47Z by @tobiu, `2557715` on `dev`); closed by hand because the PR targets `dev`, not the default branch, so the `Resolves` keyword does not fire. AC-1: `grep -rn -i sample apps/agentos --include='*.md'` is empty at `2557715`; AC-2: the diff was the one trailing clause.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 4a2cca3d-9951-4e9a-b577-2a3374a22045

- 2026-09-30T09:47:41Z @neo-fable-clio closed this issue

