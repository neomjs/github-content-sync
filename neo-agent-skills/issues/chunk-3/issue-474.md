---
id: 474
title: Align sunset and recovery skills with auto-read get_message
state: OPEN
labels:
  - documentation
  - enhancement
  - good first issue
  - help wanted
  - ai
  - model-experience
  - agent-os
assignees: []
createdAt: '2026-11-25T20:00:46Z'
updatedAt: '2026-11-25T20:00:46Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/474'
author: neo-gemini-julia
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
# Align sunset and recovery skills with auto-read get_message

Please comment to claim before starting; check current assignees and linked PRs.

## Context

`neomjs/neo-agent-brain#1925` (`63e19a3f`, resolving `neomjs/neo-agent-brain#1900` and live on the canonical Memory Core plane at `191aaaa7` per `neomjs/neo-agent-brain#1927`) landed two mailbox ergonomics changes at the `neo-mjs-memory-core` MCP boundary:
1. Recipient `get_message` calls automatically stamp `readAt` and return `readReceipt: {status: 'read', durable: true}` (unless `keepUnread: true` or `observer` / sender-only read), eliminating manual per-ID `mark_read` calls after reading direct messages.
2. `mark_read({all: true, includeUnseenBroadcasts: true})` drains unread `DELIVERED_TO` broadcasts at session end regardless of `seenAt` while preserving unseen `SENT_TO` direct messages (`withheldUnseenCount`).

`neomjs/neo-agent-brain#1900` explicitly deferred updating the skill runbooks until the Brain surface was live on `dev`.

## The Problem

Two skill payloads in `neomjs/neo-agent-skills` still describe pre-`#1925` mailbox mechanics:
- `.agents/skills/session-sunset/references/session-sunset-workflow.md` (`Step 7: Inbox Cleanup (mark_read)`, lines 119–120) instructs agents to call `mark_read` on processed inbox items without distinguishing unseen broadcasts (`includeUnseenBroadcasts: true`) from unseen direct messages that must survive into the next session, and warns against mid-task `mark_read` without noting that `get_message` now auto-stamps read DMs.
- `.agents/skills/context-recovery/references/context-recovery-workflow.md` (`Step 1` & `Step 1a`, lines 36–47) instructs calling `get_message` on each `unreadDirectMessageIds` entry and mentions bulk `mark_read` without noting that `get_message` itself stamps those read (no follow-up `mark_read` by ID needed, and `keepUnread: true` is the opt-out when leaving an unhandled DM unread).

## The Architectural Reality

- `.agents/skills/session-sunset/references/session-sunset-workflow.md` (`lines 119-121`)
- `.agents/skills/context-recovery/references/context-recovery-workflow.md` (`lines 36-47`)
- `.agents/skills/create-skill/references/skill-authoring-guide.md` (`lines 112-120`: *Tool Mechanics Live in the Tool Description, Not the Skill* — cite behavior and when-to-use rather than duplicating OpenAPI parameter schemas, and respect `maxPositiveDeltaBytes` / `npm run lint` byte budgets).
- Design authority: "For direct messages, the recipient's turn-start mailbox check is the delivery point; unseen broadcasts may be marked read by a session-end drain (`mark_read({all: true, includeUnseenBroadcasts: true})`, `#1900`), since claim sweeps query `status: 'all'`." (`neomjs/neo-agent-brain` `learn/agentos/decisions/0002-phase3-wake-substrate-standards-alignment.md:360`).

Decision Record impact: aligned-with `neomjs/neo-agent-brain#adr:0002-phase3-wake-substrate-standards-alignment` and `neomjs/neo#adr:0008-skill-anatomy-and-authoring-contract`.

## The Fix

1. Update `.agents/skills/session-sunset/references/session-sunset-workflow.md` Step 7 to state concisely that read DMs are already stamped by `get_message` (keep unhandled 1:1 DMs unread via `keepUnread: true`) and session-end cleanup uses `mark_read` with `all: true` and `includeUnseenBroadcasts: true` so unseen broadcasts drain while unseen direct messages stay unread for the successor session (preserving Step 8's continuity self-DM unread by running Step 7 before Step 8).
2. Update `.agents/skills/context-recovery/references/context-recovery-workflow.md` Step 1 to note concisely that `get_message` on `unreadDirectMessageIds` stamps read automatically (no per-ID `mark_read` call; pass `keepUnread: true` only when deferring an unhandled 1:1 DM).
3. Trim redundant surrounding prose so `npm run lint` (`scripts/lint-skill-corpus.mjs`) and `npm test` pass within `perFilePayloadBudget` and `maxPositiveDeltaBytes` without modifying top-level `SKILL.md` routers unless needed for regenerated mirrors (`npm run sync` / parity checks).

## Acceptance Criteria

- [ ] `.agents/skills/session-sunset/references/session-sunset-workflow.md` Step 7 directs session-end broadcast cleanup via `mark_read({all: true, includeUnseenBroadcasts: true})` before Step 8's continuity ping while preserving unseen/unhandled 1:1 direct messages.
- [ ] `.agents/skills/context-recovery/references/context-recovery-workflow.md` Step 1 reflects automatic `get_message` read-stamping (no redundant per-ID `mark_read`; `keepUnread: true` for deferred unhandled DMs) without re-documenting OpenAPI internals.
- [ ] `npm run lint` and `npm test` pass cleanly in `neomjs/neo-agent-skills` (including Claude plugin / mirror parity if touched payloads project there).

## Out of Scope

- Changing `neo-mjs-memory-core` runtime behavior (`MailboxService.mjs` / `openapi.yaml`), already delivered in `neomjs/neo-agent-brain#1925`.
- `list_messages({projection: 'counts'})` payload capping (`neomjs/neo-agent-brain#1555`).

## Related

- `neomjs/neo-agent-brain#1900` (originating Brain ticket)
- `neomjs/neo-agent-brain#1925` (Brain implementation PR)
- `neomjs/neo-agent-brain#1927` (post-reload live verification owner)

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Live latest-open + closed sweep: checked latest 25 open (#446..#61) + open and closed search (`get_message OR includeUnseenBroadcasts OR 1900 OR 1925`, Out of Scope / Avoided Traps read on #388, #326, #312) at 2026-11-25T19:59Z; no equivalent or prior exclusion found (43 open — orange band).
A2A in-flight claim sweep: `list_messages({status: 'all', projection: 'compact', limit: 20})` at 2026-11-25T19:59Z; no competing claim.
MC + KB rationale sweep: `query_raw_memories` (`session-sunset Step 7 context-recovery get_message auto-read includeUnseenBroadcasts`) + `ask_knowledge_base` (`type: 'ticket'`); no duplicate ticket.
Own-assignment sweep: 0 open in `neomjs/neo-agent-skills`; 3 open in `neomjs/neo-agent-brain` (#1927 owns filing this follow-up).
Structure map: `npm run ai:structure-map -- --files --loc` run; target payloads live in `.agents/skills/{session-sunset,context-recovery}/references/` (no new `.mjs` file).

Origin Session ID: 05580ab8-2f98-4b02-891c-75d444d212ec
Retrieval Hint: "session-sunset Step 7 context-recovery get_message keepUnread includeUnseenBroadcasts 1900"

## Timeline

- 2026-11-25T20:00:47Z @neo-gemini-julia added the `documentation` label
- 2026-11-25T20:00:48Z @neo-gemini-julia added the `enhancement` label
- 2026-11-25T20:00:48Z @neo-gemini-julia added the `good first issue` label
- 2026-11-25T20:00:48Z @neo-gemini-julia added the `help wanted` label
- 2026-11-25T20:00:49Z @neo-gemini-julia added the `ai` label
- 2026-11-25T20:00:49Z @neo-gemini-julia added the `model-experience` label
- 2026-11-25T20:00:49Z @neo-gemini-julia added the `agent-os` label
- 2026-11-25T20:01:30Z @neo-gemini-julia cross-referenced by #1927

