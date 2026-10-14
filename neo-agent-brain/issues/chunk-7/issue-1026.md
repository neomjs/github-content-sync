---
id: 1026
title: Seat model guard fires only when another seat shares the account
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-14T14:59:24Z'
updatedAt: '2026-10-14T14:59:24Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1026'
author: neo-opus-vega
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
# Seat model guard fires only when another seat shares the account

Sub of #571. Follow-up to #1016 (PR #1024, merged 2026-10-14).

## Context

`seatModelGuardHook` blocks a Claude Desktop prompt while the session's model family differs from the family the seat last ran. It exists because seats that share a Claude account open on each other's last picker choice. Design authority: the operator's challenge on 2026-10-14, after merging #1024, that at least one seat switches between Opus and Fable every week on purpose, on its own flatrate. For that seat each switch is deliberate, and #1024 blocks the first prompt after every one of them until the operator resends with `model-ok`. That is an operator-facing ritual for a case the guard was never meant to cover.

## The Problem

The guard's expectation, "the family this seat last ran", treats every family change as suspect. A change can only come from another seat when another seat shares the account. On an account no other seat uses, the remembered pick is always this seat's own, so every change is deliberate and blocking it is pure friction.

## The Architectural Reality

- Each Desktop seat's user-data dir is `<agentsRoot>/<agentId>/harness/claude-desktop` (Fleet's instance home). Its sessions live in `claude-code-sessions/<accountUuid>/<…>/local_<id>.json`. The guard already finds that record through `CLAUDE_CODE_EXECPATH` and `CLAUDE_CODE_HOST_SESSION_ID` (`ai/scripts/lifecycle/hooks/claude/seatModelGuardHook.mjs`).
- So the account is the first path segment of the session's own record, and "another seat shares it" means a sibling seat has a `claude-code-sessions/<sameAccount>` folder. That was verified read-only on 2026-10-14: the observed sharing matched the recorded wrong starts, and the one seat without a sharer never started wrong.

## The Fix

In `seatModelGuardHook.mjs`:
- the record read also returns the account folder name;
- `prompt` blocks only when some other seat under the same agents root has a `claude-code-sessions/<account>` folder;
- the check reads directory names only;
- an unrecognized layout (not `<agentsRoot>/<agentId>/harness/claude-desktop`) never blocks, per the guard's fail-open contract.

`stop` keeps recording the last-run family. No configuration, no projector change.

## Acceptance Criteria

- [ ] AC-1: a seat whose account folder exists under no other seat is never blocked, even on a family change.
- [ ] AC-2: a seat whose account folder also exists under another seat keeps #1024's behaviour: it blocks on a family mismatch, `model-ok` passes, and malformed input fails open.
- [ ] AC-3: an unrecognized user-data layout never blocks.
- [ ] AC-4: the sharing check reads only directory names under other seats, never their files.
- [ ] AC-5 (post-merge): a seat alone on its account switches families without a block, and a shared-account mismatch still blocks. Observed under #571 alongside #1016's AC-7.

## Out of Scope

A declared per-seat model (the #700 / #559 path) · changing which seats share accounts · the guard's message and `model-ok` semantics.

## Avoided Traps

- **A per-seat opt-out flag or file:** it asks the operator to configure something by hand, which FM should never require. The sharing fact is already on disk.
- **Reading other seats' session records:** names are enough; contents would widen what one seat reads of another.
- **The handle as the expected model:** deliberate model days exist on any seat.

## Related

#571 (parent) · #1016 / PR #1024 (the guard) · #700, Institution #559 (declared model)

Decision Record impact: none.

Live latest-open sweep: latest 20 open Brain issues at 2026-10-14T14:58:37Z, no equivalent. A2A sweep (last 15, all read states): no claim on this scope. MC sweep ("seat switches Opus and Fable weekly … guard … shared account"): no prior decision. Own-assignment sweep: no overlap. Structure map: N/A, an in-place edit of the existing hook.

Origin Session ID: 44625315-94b1-40ee-9302-6d4183995dd2

Retrieval Hint: "seat model guard shared Claude account solo seat weekly Opus Fable switch claude-code-sessions account folder"

## Timeline

- 2026-10-14T14:59:25Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-14T14:59:26Z @neo-opus-vega added the `enhancement` label
- 2026-10-14T14:59:26Z @neo-opus-vega added the `ai` label
- 2026-10-14T14:59:27Z @neo-opus-vega added the `agent-os` label
- 2026-10-14T15:01:52Z @neo-opus-vega cross-referenced by PR #1027

