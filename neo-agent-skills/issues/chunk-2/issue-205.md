---
id: 205
title: 'Direct messages never suppressed: retire peer-role''s ack carve-out'
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-fable-mnemosyne
createdAt: '2026-10-24T15:20:16Z'
updatedAt: '2026-10-24T15:20:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/205'
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
# Direct messages never suppressed: retire peer-role's ack carve-out

## Context

Operator directive, 2026-10-24 (report, in chat, to several seats): *"super important not only for today: 1:1 wake messages. otherwise other peers that idled out might miss messages."* The mechanical side — the Memory Core guard that today accepts a quiet direct message unless its subject is known-actionable — is neomjs/neo-agent-brain#1250. This ticket is the rule's echo in the skills every seat loads, because the skills currently *teach* the carve-out the operator retired.

The specimen: two acks sent quiet to peers on 2026-10-24 by a seat with the skill loaded (MESSAGE:f02e1282 to Sophie, MESSAGE:42acdab4 to Vega), both unread while Sophie's seat idled. An idled-out seat has no mailbox poll; only the wake route brings it back.

## The Problem

`peer-role/references/peer-role-mode.md:122` (§ Wake-control) says: *"Direct messages wake by default — that is where actionability lives … Suppress a direct message only for non-overlapping awareness — observer notes, lane-progress, acks; suppressed actionable direct lifecycle subjects are rejected mechanically."* The sentence makes the sender judge the recipient's attention. Under the operator's rule there is no such judgement for 1:1 traffic: the address decides — a peer address wakes, `AGENT:*` is quiet.

Two other skill passages suppress targeted messages on purpose and must be re-read against the rule, not silently kept:

- `session-sunset/references/session-sunset-workflow.md:144,162` — the continuity **self-DM** (`to === sender`, tag `sunset-protocol-handover`) stays quiet so the shutting-down session is not re-woken. The rule is about *other* peers; this exception stands and should say so explicitly.
- `lead-role/references/lead-role-mode.md:172` — the **baton** is a targeted DM to the new lead with `wakeSuppressed: true` (the Brain's allow-list `WAKE_SUPPRESSION_ALLOWED_TAGS = ['sunset-protocol-handover', 'lead-role-baton']` admits it). The new lead is exactly a seat that may have idled out; neo-agent-brain#1250 recommends the baton wakes. The skill follows whatever that review decides.

## The Architectural Reality

Three reference files, no SKILL.md router touched (Progressive Disclosure kept; `create-skill` consulted). The engine's `learn/agentos/AGENTS_ATLAS.md:144` repeats the baton's `wakeSuppressed: true` — a different repository, one line, filed once the baton decision lands (Out of Scope here). `ticket-create-workflow.md:42` already carries the complementary rule for broadcasts (a contested-lane resolution MUST set `wakeSuppressed: false`) and needs no change.

**Design authority:** the operator's directive of 2026-10-24; it supersedes the awareness carve-out that `#15987`'s wake-control text introduced for direct messages (its broadcast side — quiet claims by default — is untouched).

## The Fix

- `peer-role-mode.md:122`: replace the carve-out with the rule — *a direct message to a peer always wakes (omit `wakeSuppressed` or set it false), whatever its content: review, ack, lane-progress, observer note; the server rejects `true` on a peer address (neo-agent-brain#1250). Quiet is for `AGENT:*` and for your own continuity self-DM.* Keep the `priority: 'high'` sentences as they are.
- `session-sunset-workflow.md:144`: add the one clause that names the exemption's reason (self-addressed; the rule for peers does not apply).
- `lead-role-mode.md:172`: baton wakes (drop `wakeSuppressed: true`) if #1250 lands that way; otherwise the line cites the named exception.
- `turn-memory-pre-flight` applies to all three edits (skill-loaded substrate). Accretion: the edit replaces sentences, it adds none net; no new file.

## Acceptance Criteria

- [ ] `peer-role-mode.md` § Wake-control no longer names any class of direct message that may be suppressed; it states the address rule and the server's rejection in two sentences.
- [ ] `session-sunset-workflow.md` names the self-DM as the exemption and why.
- [ ] `lead-role-mode.md:172` matches the baton decision recorded in neo-agent-brain#1250's guard JSDoc.
- [ ] A grep for `wakeSuppressed: true` across `.agents/skills/**` returns only the self-DM (and the baton line if the exception is kept).
- [ ] Loaded-bytes of the three files do not grow (`wc -c` before/after in the PR body).

## Out of Scope

The Memory Core guard, `openapi.yaml` and `A2A.md` (neo-agent-brain#1250); the engine's `AGENTS_ATLAS.md:144` baton line (its own one-line ticket after the baton decision); any new tag vocabulary for direct messages.

## Related

neomjs/neo-agent-brain#1250 (the mechanical rule, same operator word); neo-agent-skills#191 (mine, unrelated); neo `#15987` (the wake-control text's origin).

**Sweeps.** Live latest-open: the 20 newest open issues of this repository read at 2026-10-24T15:16Z (newest #191) — none on wake control; the Brain's 20 newest read in the same minute — #1250 is the sibling filed a minute ago. A2A in-flight (`list_messages` all, 30, 15:16Z): Ada's (15:13Z) and Clio's (15:15Z) lines relay the rule, no claim on the skill text; my `[lane-intent]` MESSAGE:97f94967 (15:14:33Z) is the earliest claim. Memory Core rationale sweep: Clio's 2026-09-23 memory on explicit wakes — consistent. Own open tickets: #191 only, no overlap. Structure map: N/A (no `.mjs`, no new file; `create-skill` router read).

Origin Session ID: eb15a38a-6ff6-4a2f-94c2-47b29e846467

Retrieval Hint: `query_raw_memories("direct message wake suppressed peer-role wake-control ack carve-out")`

— Mnemosyne · @neo-fable-mnemosyne · Claude Fable 5.1 · Claude Code

## Timeline

- 2026-10-24T15:20:16Z @neo-fable-mnemosyne assigned to @neo-fable-mnemosyne
- 2026-10-24T15:20:17Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-10-24T15:20:18Z @neo-fable-mnemosyne added the `ai` label
- 2026-10-24T15:20:18Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-10-24T15:21:25Z @neo-fable-mnemosyne cross-referenced by #1250

