---
id: 248
title: Honor explicit goal-mode requests before ending the turn
state: OPEN
labels:
  - bug
  - ai
  - model-experience
assignees: []
createdAt: '2026-10-30T19:08:28Z'
updatedAt: '2026-10-30T19:08:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/248'
author: neo-gpt-emmy
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
# Honor explicit goal-mode requests before ending the turn

## Context

On 30 October the operator asked about restarting native goal modes and renewed Stage lead coordination. Emmy repeatedly observed `goal:null` and ended with “no goal activated” proposals, without resolving the interpretation through a focused clarification or continuing the already-authorized work. The operator then explicitly corrected this as a missed continuation request and reported idle GPT peers and cold-cache cost.

After that correction, Emmy created an ACTIVE goal at 18:47Z; Euclid and Sophie independently reported ACTIVE goals at 18:53/18:54Z, and Euclid produced a native artifact at 18:58Z. This is an observed handoff/activation failure, not evidence the harness goal tools failed.

Defect capture: `MESSAGE:9f23c961-0e4f-4cbd-a5fa-40acebf20a65`, fingerprint `377567ef76c1608b`.

## The Problem

A well-scoped plan can terminate at prose while the requested sustained execution never starts. “Proposed” and “sent” do not establish either the local goal state or a peer's resumed work. Genuine ambiguity must be resolved once rather than preserved across repeated proposal-only turns.

## The Architectural Reality

Design authority: the native Codex goal tool says, “Create a goal only when explicitly requested by the user or system/developer instructions; do not infer goals from ordinary tasks.”

The existing `.agents/skills/goal-scoping/references/goal-scoping-workflow.md` defines observable outcomes and self-selected lanes. `.agents/skills/lead-role/references/lead-role-mode.md` requires the lead to take its own lane. These govern the handoff decision; the harness owns tool availability, permitted state transitions and mechanics.

The create-skill guide's “Tool Mechanics Live in the Tool Description” rule excludes copied parameters or a second tool manual. Brain neomjs/neo-agent-brain#1373 owns delivery/actuation evidence; dispatch does not establish recipient goal activation. Skills #247 separately owns Claude context/cache waiting.

## The Fix

Add a short conditional discipline at the existing goal-scoping/lead handoff decision: resolve an explicit native-goal continuation request into the actual available harness state and begin the first independent authorized work in the same turn. Preserve existing goals and use the harness's own permitted transition semantics.

For genuine ambiguity, ask one focused clarification while advancing work already authorized and independent of that answer. Missing/refused capability is reported as such. A user-authorized peer continuation gets a direct waking handoff, with recipient activation reported only from its acknowledgement. Do not turn this into a universal goal-creation default, new skill router, or always-loaded checklist.

## Contract Ledger

| Target surface | Authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Existing goal-scoping/lead handoff payload | Native goal tool descriptions and explicit user intent | Resolve authorized continuation to observed state plus first work | Clarify true ambiguity once; report unavailable/refused accurately | Small conditional in existing payload; no parameter copy | Explicit/ambiguous/no-authorization/existing-goal replay |
| Peer handoff wording | Existing direct A2A semantics | Distinguish sent, resumed turn and recipient-confirmed goal state | Unknown remains unknown | Existing handoff prose | One actual recipient acknowledgement; no sender-only activation claim |

## Acceptance Criteria

- [ ] An explicit actionable native-goal request from no goal results in an actual ACTIVE state and first authorized action in the same turn; reporting matches the readback.
- [ ] An unfinished goal is not silently replaced. Resume follows the harness's permitted semantics; refusal or missing capability is reported accurately.
- [ ] Genuine ambiguity gets one focused clarification while independent authorized work proceeds; repeated proposal-only endings do not count as resolution.
- [ ] An ordinary task, question or autonomous wake without explicit user/system goal authorization creates no new goal.
- [ ] User-authorized peer continuation uses direct waking handoffs and distinguishes delivery from recipient-confirmed activation. No unavailable peer state is invented.
- [ ] A fresh or compacted session exercises the correction. Keep a small behavioral replay of the cases above and run the repository's skill lint/byte checks; do not present discipline as mechanical enforcement.

## Out of Scope

New daemon, scheduler, ledger or coordination platform; changing native goal tool schemas; automatic goal creation for ordinary work; imposing Codex tools on unsupported harnesses; duplicating #247 or Brain's transport repair.

## Avoided Traps and Decision Record Impact

Do not weaken the explicit-request boundary to fix missed activation. Do not replace named peers' self-selection with worker assignment. Rewrite the smallest existing conditional; avoid new top-level instruction bytes. Aligned with ADR 0008's progressive-disclosure boundary; no runtime placement or representation change.

## Creation Evidence and Ownership

Live latest-open sweep at 2026-10-30 19:08Z requested 20 and returned all 18 open Skills issues; #61 concerns missed skill triggers, #247 concerns Claude cache/context waiting, neither covers native-goal activation. Latest 30 all-state A2A messages contain the incident and current goal handoffs, with no competing exact-scope claim. Symptom-keyed MC search returned three mostly unrelated results; no inference of absence is made from it. Own-assignment body read found only unrelated #20. Labels verified. The scoped Brain structure-map command ran successfully on the existing goal-scoping folder and identified its router and reference payload; no new runtime module is proposed.

unowned-rationale: captured separately so the operator-escalated correction survives while Emmy continues the active Stage production goal; a Skills maintainer can self-select this bounded one-PR repair.

Related: #61, #247, neomjs/neo-agent-brain#1373, neomjs/neo-stage#522.

Origin Session ID: 836d53ea-6ee2-4dc8-9791-c2e55f913667
Retrieval Hint: "requested goal mode idle peers cold cache no progress proposal-only handoff"

## Timeline

- 2026-10-30T19:08:30Z @neo-gpt-emmy added the `bug` label
- 2026-10-30T19:08:30Z @neo-gpt-emmy added the `ai` label
- 2026-10-30T19:08:30Z @neo-gpt-emmy added the `model-experience` label

