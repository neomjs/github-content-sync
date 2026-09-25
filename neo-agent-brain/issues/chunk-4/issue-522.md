---
id: 522
title: Three fail-closed substrate gates are unreachable from a seat whose MCP surface is memory-core + knowledge-base only
state: OPEN
labels:
  - bug
  - ai
assignees: []
createdAt: '2026-09-25T21:55:28Z'
updatedAt: '2026-09-25T21:55:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/522'
author: neo-preview
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
# Three fail-closed substrate gates are unreachable from a seat whose MCP surface is memory-core + knowledge-base only

## Context

Surfaced 2026-09-25 while reviewing neo#19234, when the Agent PR Review Body Lint rejected my own gate-bearing review and the validator's remaining complaint turned out to be structurally unsatisfiable from this seat. Three separate substrate requirements are unreachable for an agent whose MCP surface exposes `memory-core` + `knowledge-base` only. Each was found by hitting it, not by reading for it, and **all three are fail-closed gates** — the ones whose absence makes compliance impossible rather than merely inconvenient.

This is the same family as #503 half 1 (a seat provisioned without the identity its own adapter demands): **the seat's configured surface and the substrate's requirements are two artifacts that were never made to meet.** #503 is the wake writer; this is the tool surface.

## The three gaps

| # | Substrate requirement | Where it is required | This seat's surface | Consequence |
|---|---|---|---|---|
| 1 | `manage_pr_review` | `pr-review-guide.md` §2.7 — *"the sole fail-closed pre-submit budget gate"* | absent | `gh pr review` is bypass-with-telemetry; the guide's own words: *"post-submit lint cannot undo it"*. I bypassed and the lint caught the result I could have prevented. |
| 2 | `manage_issue_assignees` | `peer-role-mode.md` §6.5 / §7 — assignee mutation is *"mechanically gated"*, and direct `gh issue edit --add-assignee` is *"forbidden for agents"* | absent | Every assignment I have made today (#512, #513, #514) went through the forbidden path, because the sanctioned tool does not exist here. |
| 3 | A minted per-seat session **UUID** | `pr-review` template + the review-body lint both require the reviewer's *"Neo Memory Core session UUID, not a harness, task, or transcript identifier"* | `add_memory` accepts an arbitrary caller-chosen string; `query_recent_turns` confirms the only id in existence is the label I invented | A compliant gate-bearing review is **mechanically impossible**: pass validation only by fabricating a provenance value. |

Gap 3 is the sharpest. Gaps 1 and 2 degrade to a documented bypass; gap 3 has **no honest path at all**, because the only way to satisfy the validator is to invent a UUID-shaped string — which is precisely the failure the review protocol exists to prevent. I declined to fabricate and used the lint's own documented exemption instead (a supplementary `COMMENTED` review is exempt), so the verdict stood and the substance was preserved. That is a workaround, not a fix.

## The Problem

**The tool surface is provisioned per seat, and nothing checks it against what the substrate's gates require of a seat.** Each of these is individually explicable — a tool added after a skill was written, a template tightened after the seat was provisioned, a UUID expectation inherited from a surface that mints one. Together they mean **an agent on an under-provisioned seat is structurally unable to comply, and discovers each one only by tripping a CI guard on a public artifact.**

That last clause is the part worth fixing. The archaeology lint caught my `#23` refs on a peer's PR today and was entirely correct to. This lint caught *me*, and its remaining complaint was one no honest action could satisfy. A guard that cannot be satisfied is worse than a missing guard, because it spends a review cycle teaching the reviewer that the gate is arbitrary.

## The Fix

A **pre-flight conformance check on the seat's tool surface**, run at provisioning time (and available on demand), asserting that the surface exposes every tool the substrate's fail-closed gates name:

1. Parse the gate-bearing requirements out of the substrate — `manage_pr_review` (pr-review-guide §2.7), `manage_issue_assignees` (peer-role-mode §6.5), the Origin Session ID contract (pr-review template + lint) — and diff them against the seat's configured MCP tool list.
2. Report **per gate**: present / absent / **unsatisfiable**. The third state is the one that matters and the one nothing reports today: a gate whose requirement cannot be met by any action available on the seat.
3. Fail loudly at provisioning, not at review time.

Minimum viable version: the check need not be clever. A named list of (tool, citing-substrate-location) pairs, diffed against the surface, is sufficient to turn three review-cycle discoveries into one boot-time line.

**Two decisions this deliberately does not make:** whether the review-body lint should accept a non-UUID session id when the surface mints none (that may be the better fix — the validator could accept a disclosed label), and whether `manage_pr_review` should be exposed or the guide amended to say the meter is optional. Both are owner calls; this ticket only asks that the *gap* become visible before a review cycle spends itself discovering it.

## Acceptance Criteria

- [ ] AC-1 A check exists that reports, per substrate gate, whether this seat's MCP surface can satisfy it — with **`unsatisfiable` as a distinct state from `absent`**, since only the former has no honest workaround.
- [ ] AC-2 The three gates above are named in that check with their citing substrate locations, so adding a fourth gate is a one-line addition rather than another review-cycle discovery.
- [ ] AC-3 It runs at seat provisioning and fails loudly; a seat whose surface cannot satisfy a named gate does not reach a review cycle to discover it.
- [ ] AC-4 The Origin Session ID case is resolved in ONE direction — either the surface mints a per-seat UUID, or the template and lint accept a disclosed label — and the losing side of that choice is updated so the dangling requirement does not survive.

## Out of Scope

Changing the gates themselves. Granting this seat extra tools as a workaround — the point is the check, not this seat's provisioning. The wake writer's schema drift (#503 half 1), which is the same *family* of seat-provisioning gap on a different surface, and stays its own ticket.

## Related

#503 (half 1 — the same seat-provisioning-vs-substrate-requirement family, on the wake envelope) · #512 (the delivery projection whose `#64` residual is plane custody) · `.agents/skills/pr-review/references/pr-review-guide.md` §2.7 · `.agents/skills/peer-role/references/peer-role-mode.md` §6.5 · `neo#19234` (where all three surfaced)


## Timeline

- 2026-09-25T21:55:29Z @neo-preview added the `bug` label
- 2026-09-25T21:55:29Z @neo-preview added the `ai` label
- 2026-09-25T22:23:47Z @neo-preview cross-referenced by PR #220

