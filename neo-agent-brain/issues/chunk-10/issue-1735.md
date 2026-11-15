---
id: 1735
title: Retire AppleScript message delivery after native peer validation
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-11-15T01:14:15Z'
updatedAt: '2026-11-15T01:14:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1735'
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
blockedBy:
  - '[ ] 1731 Fleet provisions native wakes for managed Codex and Antigravity seats'
blocking: []
---
# Retire AppleScript message delivery after native peer validation

## Context

Design authority: the operator elected a one-time cut after signed native delivery works for our real peers. This is the retirement step for the normal Fleet setup in #1731. Existing operators do not need a permanent legacy-mode switch.

## The Problem

Removing the old route before both real peer journeys are proven would strand running seats. Keeping the old sender after they are proven would leave supported clients able to type a peer message into a desktop composer, including after a future configuration regression. Codex has a confirmed installed wake. The signed Antigravity real-recipient witness is still pending on #571.

## The Architectural Reality and Fix

Fleet chooses ordinary wake transport in `armFleetSeatWake`. The receiver accepts historical signed snapshots in `receiver.mjs` and dispatches through `localWakeAdapters.mjs`. `daemon.mjs` also has a direct AppleScript sender. Migrate durable unmatched records once by subscription/seat and preserve proven receipt and unknown-send semantics. Remove window typing and its selection/defaults for A2A digests in both paths. Inspect subscribers and manifest readers, templates and diagnostics against the emitted supported contract. Preserve the independent authenticated Claude `session/rotate` command and the existing non-typing OpenCode transport. Evaluate OpenCode separately with evidence if its use resumes.

| Target | Source | Behavior after cut | Verification |
| --- | --- | --- | --- |
| Signed `wake/digest` / direct send | Family-native transport and durable mailbox | No path writes peer text, simulates Enter, creates a new chat, or downgrades a native record to GUI typing | Trace both dispatch entry points, tampered/missing route, ambiguous send, startup/handoff and retry |
| Existing operator’s fleet | Installed receipt and authenticated seat bindings | Convert remaining eligible seats once, preserve old queued work without duplicates, leave foreign/private profiles alone | Reconcile selected root, no subagent delivery, current-epoch/model receipt and per-seat migration results |
| Explicit `session/rotate` | Separate signed rotation route | A deliberate Claude sunset still opens its one successor | Existing isolated rotation suite plus installed sunset receipt |

## Acceptance Criteria

- [ ] Before removal, attach installed signed receipts for real Codex and Antigravity main chats on the provisioned paths, with focus/draft observations, old-root/helper rejection and queue disposition. Keep the operational legacy lane until this evidence lands.
- [ ] Default Add/Start for a new managed Codex or Antigravity seat selects native delivery without an enrollment CLI or any global Codex setting; an unready seat keeps its mail pending with a visible reason.
- [ ] Remove A2A typing senders/defaults and tests that bless them, replacing them with negative controls: no typing dispatch under missing/mismatch/retry, and one native delivery after recovery. Claude hook delivery and the separate sunset command retain positive controls. No permanent legacy preference.
- [ ] Record one migration for the current fleet and repeat the fresh-operator installation smoke with independent seats, including coexistence with an unmanaged Codex instance. Installed owner: #571.

Out of scope: OpenCode changes without a real owner/use case; removing osascript uses outside A2A message delivery.
Decision Record impact: aligned with the existing signed wake/rotation split. Related: #1731 · #1706 · #562 · #768 · #1373. Not a newcomer ticket: execution waits for the protected installed recipients.
Live latest-open + closed sweep: newest 20 Brain issues, red backlog; AppleScript titles #30, #562, #606, #768 are earlier Claude migration or legacy reliability work. #1731 owns preparation. No competing retirement claim in the recent 30 A2A rows.
Origin Session ID: 1188c53b-f31f-4f66-a94b-fc696eb02810
Retrieval Hint: "retire osascript peer wake digest after real Codex and Antigravity receipt"

## Timeline

- 2026-11-15T01:14:17Z @neo-gpt-emmy added the `enhancement` label
- 2026-11-15T01:14:17Z @neo-gpt-emmy added the `ai` label
- 2026-11-15T01:14:17Z @neo-gpt-emmy added the `agent-os` label
- 2026-11-15T01:15:18Z @neo-gpt-emmy cross-referenced by #1731
- 2026-11-15T01:16:59Z @neo-gpt-emmy marked this issue as being blocked by #1731
- 2026-11-15T01:25:35Z @neo-gpt-emmy cross-referenced by PR #1737
- 2026-11-15T01:29:03Z @neo-gpt-emmy cross-referenced by #1738

