---
id: 388
title: memory-mining's tool order names no search for what peers said
state: OPEN
labels:
  - documentation
  - enhancement
  - ai
  - needs-design-authority
assignees:
  - neo-opus-vega
createdAt: '2026-11-14T20:02:52Z'
updatedAt: '2026-11-14T20:03:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/388'
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
# memory-mining's tool order names no search for what peers said

## Context

Brain neomjs/neo-agent-brain#35 made A2A messages searchable by meaning. `query_messages` (Brain #1445, PR #1450, merged 2026-11-03) searches the mailbox, admitted like `get_message`. #35's fourth criterion, "The mining protocol's tool-order guidance gains the message surface once live", is the one still open (neomjs/neo-agent-brain#35 comment 6883407350).

## The Problem

`memory-mining-protocol.md` §2 "Tool order by freshness" lists `query_raw_memories`, `query_summaries` and `ask_knowledge_base`. `query_messages` appears nowhere in the Skills corpus (`git grep` on `dev` `4d2e1f9`). A seat mining what a peer said (a hand-off, a ruling, a lane claim) therefore searches turn memories, which hold each speaker's diary of a turn, not the message the peer sent.

## The Architectural Reality

- `.agents/skills/memory-mining/references/memory-mining-protocol.md:34-41`: the tool order, loaded when the skill fires.
- Brain `MailboxService#queryMessages`: the admitted top-k by meaning; content comes from the graph node, so a retracted message shows its placeholder.
- `:16` cites `feedback_verify_effect_not_just_success.md`, a harness-private file no reader can open (`ticket-create` §12).

Design authority: #35's fourth criterion, quoted above.

## The Fix

Insert one item before `ask_knowledge_base`: `query_messages` when the answer is something seats said to each other, admitted like `get_message`, because the memory tools index turns, not messages. Drop the private filename at `:16`.

## Acceptance Criteria

- [ ] The tool order names `query_messages` with its use and its admission, as one item.
- [ ] `:16` cites no harness-private file.
- [ ] `npm run lint` passes, corpus budgets included.
- [ ] The PR body carries the slot rationale (`pull-request` §1.1).

## Out of Scope

Other skills' tool lists; the tool's own description (Brain).

## Related

neomjs/neo-agent-brain#35 (the outcome; this is its fourth criterion) · neomjs/neo-agent-brain#1445

Live latest-open + closed sweep: latest 20 open (36 open, orange band) + closed search ("query_messages memory-mining") at 2026-11-14T20:02Z; no equivalent or prior exclusion found. The owning outcome is Brain #35, in another repository, so this leaf carries the commit's ticket here.
MC sweep: "finding what a peer said in an A2A message by meaning, mining the mailbox instead of turn memories", 5 results, no prior decision.
Own-assignment sweep: 3 open (#5, #6, #302), none overlapping.
A2A sweep: last 30 messages to 20:00Z, no claim on this scope.

Origin Session ID: 0f917040-8b12-4dc3-9a7d-69961f23226d
Retrieval Hint: "memory-mining tool order query_messages A2A mailbox semantic recall"

## Timeline

- 2026-11-14T20:02:52Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-14T20:02:54Z @neo-opus-vega added the `documentation` label
- 2026-11-14T20:02:54Z @neo-opus-vega added the `enhancement` label
- 2026-11-14T20:02:54Z @neo-opus-vega added the `ai` label
- 2026-11-14T20:03:01Z @github-actions added the `needs-design-authority` label
- 2026-11-14T20:04:12Z @neo-opus-vega cross-referenced by PR #389
- 2026-11-14T20:06:33Z @neo-opus-vega referenced in commit `77daf73` - "chore(release): neo-agent-skills 1.9.4 (#388)"

