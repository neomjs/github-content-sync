---
id: 203
title: 'context-recovery reads a repository''s VISION.md, when present, before it asserts lane state'
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-fable-mnemosyne
createdAt: '2026-10-23T12:14:09Z'
updatedAt: '2026-10-23T21:04:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/203'
author: neo-fable-mnemosyne
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
closedAt: '2026-10-23T21:04:04Z'
---
# context-recovery reads a repository's VISION.md, when present, before it asserts lane state

## Context

neo-stage#321: the operator asked for a Stage `VISION.md` that peers do not ignore in fresh sessions or after compaction. The Stage repository's own `AGENTS.md` now orders every harness to read it at boot (neo-stage#322); the engine's `AGENTS.md` gets a Stage trigger line (neomjs/neo, filed beside this). The third load path is this skill: a seat that recovers after compaction runs `context-recovery` and reconstructs its lane from the Memory Core — the runbook names mailbox, recency, semantic anchors, session rollups and the live substrate, and no repository direction page.

## The Problem

A post-compaction seat can resume a lane whose direction the repository has since changed (the sculpted route at Stage was re-entered this way after resets). The runbook's step 5 ("live substrate check") verifies tickets, PRs and branches; it does not read what the repository says the work is for.

## The Architectural Reality

`.agents/skills/context-recovery/references/context-recovery-workflow.md` §3 (the recovery sequence) and §4 (the ledger). Progressive disclosure: the router `SKILL.md` stays a pointer; the payload grows by one step and one ledger line. Every consumer repository pins this package.

## The Fix

In §3, a step 5a: "**Direction check:** if the checked-out repository carries a `VISION.md`, read it before asserting lane state; a lane that contradicts it is not resumed, it is re-planned against the roadmap." In §4's ledger a line `direction: <VISION.md read | none in this repository>`. Net loaded bytes: about 300; sunset: retire when the harness loads the page itself (the Stage `AGENTS.md` route) in every harness, measured by boot receipts.

Decision Record impact: none.

## Acceptance Criteria

- [ ] AC-1 the step and the ledger line are in the payload; `SKILL.md` unchanged.
- [ ] AC-2 the PR's slot rationale states the byte delta and the sunset condition.
- [ ] AC-3 a consumer pin bump follows the release (the usual Dependabot path, #144).

## Out of Scope

The Stage pages; the engine trigger line.

## Related

neo-stage#321, neo-stage#322; #61 (skill triggers measurably not firing) is adjacent: this line fires inside a skill that compaction already invokes.

Sweeps: live latest-20 open at 2026-10-23T12:13Z — #61 adjacent, not equivalent; A2A claims: none; own-assignment (skills): #191. Structure map: N/A.

Origin Session ID: 4cd05fa5-c51e-4328-a717-13074787d569
Retrieval Hint: "context-recovery VISION.md direction check before lane state"

🪢 Mnemosyne (Claude Fable 5.1 · Claude Code) · session 4cd05fa5-c51e-4328-a717-13074787d569

## Timeline

- 2026-10-23T12:14:09Z @neo-fable-mnemosyne assigned to @neo-fable-mnemosyne
- 2026-10-23T12:14:11Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-10-23T12:14:11Z @neo-fable-mnemosyne added the `ai` label
### @neo-fable-mnemosyne - 2026-10-23T20:26:32Z

### Contract Ledger (T3) — the direction check in the recovery runbook

Posted on the source ticket before the PR opens (the Contract-Completeness audit reads the originating ticket). Both surfaces are **new**; the existing surfaces they attach to were verified at `neo-agent-skills` dev `498a96c`: `context-recovery-workflow.md` §3 step 5 (lines 63–65) and the §4 ledger block (lines 106–116); `VISION.md` exists in `neo-stage` at `bc55e52` (33 lines; its first line asks for the read at every boot and after every compaction) and does not exist in `neomjs/neo`.

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `context-recovery-workflow.md` §3 step **5a** (new) | #203; neo-stage#321 / neo-stage#322 (the Stage `AGENTS.md` boot route reads the page; this runbook was the third load path with no direction read) | After the live-state check and before any lane-state assertion, the recovering seat reads the checked-out repository's `VISION.md`; a lane that contradicts it is re-planned against the roadmap, not resumed | No `VISION.md` in the checkout → the step is a no-op recorded as `none in this repository`; a direction page under another name is out of scope (the engine trigger line is #203's Out of Scope) | The payload is the doc; `SKILL.md` unchanged | L1: the diff + the corpus lint with the growth marker; L3: the first recovery of a consumer on the released pin shows the ledger line (`VISION.md read` on a Stage seat, `none in this repository` on the engine) |
| `context-recovery` ledger line `direction:` (new) | #203 | §4's ledger gains `- direction: <VISION.md read \| none in this repository>` between `live-state` and `confidence` | A ledger without the line means an older pin, not a failed check — the runbook versions by the package pin (AC-3) | same | same as above |

Sunset (both rows): retire when every harness loads the page itself through the repository `AGENTS.md` route, measured by boot receipts.

🪢 Mnemosyne (Claude Fable 5.1 · Claude Code) · Memory Core session b9535c15-9729-4bef-a8d4-c80998f1fc69


- 2026-10-23T20:27:01Z @neo-fable-mnemosyne cross-referenced by PR #204
- 2026-10-23T20:28:11Z @neo-fable-mnemosyne referenced in commit `7f6dac1` - "chore(release): 0.1.57 (#203)"
- 2026-10-23T21:04:04Z @tobiu referenced in commit `d39d58f` - "feat(skills): the recovery runbook reads a repository's VISION.md before asserting lane state (#203) (#204)

* feat(skills): the recovery runbook reads a repository's VISION.md before asserting lane state (#203)

[skill-growth-justified: 259 bytes in the conditional payload, zero in the router; the step retires when every harness loads the page itself through the repository AGENTS.md route, measured by boot receipts]

* chore(release): 0.1.57 (#203)"
- 2026-10-23T21:04:05Z @tobiu closed this issue

