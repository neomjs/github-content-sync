---
id: 1556
title: The Codex context hook names the full-size mailbox call
state: OPEN
labels:
  - enhancement
  - good first issue
  - ai
  - performance
assignees: []
createdAt: '2026-11-10T00:00:01Z'
updatedAt: '2026-11-10T00:00:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1556'
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
# The Codex context hook names the full-size mailbox call

## Context

neomjs/neo-agent-skills#326 found that the loaded rules call `list_messages({status:'unread'})` at its full-size default, and neomjs/neo-agent-skills#328 changes the rule text to `projection: 'counts'`. One more copy of the call lives outside the skills package, in a hook this repository owns.

## The Problem

`ai/scripts/lifecycle/hooks/codex/codex-context.mjs:244` defines `CORE_PREFLIGHT` as "Core preflight: call `list_messages({status:'unread'})` now…". A Codex seat that follows it loads a page of 50 full rows (36 KB measured on 2026-10-24, #1254) where the `counts` projection answers the same question in about 4 KB at 1,277 unread, and under 1.5 KB once #1555 lands. After #328 the hook and section 1500 of `AGENTS.md` will name two different calls for the same check.

## The Architectural Reality

- The string is a constant in the Codex context hook; nothing computes it.
- `projection: 'counts'` returns `unreadCount` and `unreadDirectMessageIds` and marks no row as seen (#1254, AC-4).
- Design authority: "The recovery runbook's mailbox step becomes: `counts` first, then `get_message` on the direct ids" (#1254, The Fix, item 2).

## The Fix

`CORE_PREFLIGHT` names `list_messages({status:'unread', projection:'counts'})`. A spec that pins the string follows it.

## Acceptance Criteria

- [ ] The hook's preflight names the `counts` projection.
- [ ] A grep over `ai/` for `list_messages({status` in the PR body shows no other literal default call outside specs.
- [ ] The hook's spec passes.

## Out of Scope

The Claude stop hook's prose (`laneStateStopHook.mjs:296`, `:389`), which names the tool without a call shape, and the size of the `counts` answer (#1555).

## Related

#1254, #1555, neomjs/neo-agent-skills#326, neomjs/neo-agent-skills#328.

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Live latest-open + closed sweep: newest open issues and a search over all states (codex-context, "Core preflight", codex preflight) at 2026-11-09T23:59Z; no equivalent found; 171 open (red): no owning ticket for hook text, and #1555 owns a different surface (the answer's size). A2A in-flight claim sweep: newest rows at 23:57Z, no claim on this scope. MC sweep: run for #1555 ("boot mailbox check costs a page of unread broadcast rows every turn"), 5 results, no prior decision. Own-assignment sweep: #1258, #95, #122, #144 here, none overlapping. Structure map: owning file `ai/scripts/lifecycle/hooks/codex/codex-context.mjs`; no new file.

Origin Session ID: 6bc6e94b-0b0c-4f41-ad1f-ec6dcdbfdf6d

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session 6bc6e94b-0b0c-4f41-ad1f-ec6dcdbfdf6d

## Timeline

- 2026-11-10T00:02:26Z @neo-fable-mnemosyne cross-referenced by PR #328

