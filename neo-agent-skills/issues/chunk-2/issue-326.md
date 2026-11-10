---
id: 326
title: The loaded rules read the mailbox and memories at full size
state: CLOSED
labels:
  - enhancement
  - ai
  - performance
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-09T23:36:28Z'
updatedAt: '2026-11-10T01:21:46Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/326'
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
closedAt: '2026-11-10T00:29:47Z'
---
# The loaded rules read the mailbox and memories at full size

## Context

On 2026-11-09 a fresh Claude Code session ran `context-recovery` while watching what the runbook loads. Four calls in the loaded rules name a Memory Core tool without its size parameters, so the tool's full-size default applies. The cheap forms exist: `projection: 'counts' | 'compact'` on `list_messages` (neomjs/neo-agent-brain#1254, shipped 2026-10-25) and `detail: 'summary'` on `query_raw_memories` (neomjs/neo-agent-brain#1280). `#1254` named this runbook change as a follow-up "filed when this lands". It was never filed. This is that follow-up.

The cost is the same in tokens on every seat and very different in share: the windows in the roster run from 1M down to 258k tokens. The operator's observation that opened this: a recovery often takes more than 100k.

## The Problem

| # | Call as written | Where | Answer as written | Cheap form | Its answer |
|---|---|---|---|---|---|
| 1 | `list_messages({status:'unread'})` | `agents-md/sections/1500-mailbox-check-protocol.md` (every turn, every seat) and `context-recovery-workflow.md` §3 step 1 | a page of 50 full rows: 36 KB measured 2026-10-24 (`neo-agent-brain#1254`), about 0.55 KB per row sampled now | `projection: 'counts'` | about 4 KB at 1,277 unread, under 1.5 KB once neomjs/neo-agent-brain#1555 lands; it lists `unreadDirectMessageIds` |
| 2 | `list_messages({box:'outbox'})` | `context-recovery-workflow.md` §3 step 1a | a page of 50 full rows | `projection: 'compact', limit: 10` | about 0.4 KB per row |
| 3 | `query_raw_memories` on 2 to 4 anchors, no parameters named | `context-recovery-workflow.md` §3 step 3 | the tool's default is `detail: 'full'`, `nResults: 10`; two sampled memories were 2.6 and 4.3 KB, so 26 to 43 KB per query | `detail: 'summary', nResults: 5` | about 0.55 KB per row |
| 4 | `list_messages({ status: 'all', limit: 30 })` | `ticket-create-workflow.md` §1a (ii) | 30 full rows | `projection: 'compact'` | about 0.4 KB per row; the sweep reads subjects only |
Sizes are read off the returned output in one session on 2026-11-09 unless a date says otherwise. By arithmetic from those sizes, a recovery run as written loads 110 to 250 KB of tool answers (roughly 30k to 70k tokens, 12 to 27 % of a 258k window). The cheap forms load 15 to 20 KB. No literal run was made. Row 1 is also paid at every turn start.

The same run showed two more inputs:

- **A restart after a sunset pays for a recovery after a compaction.** The handover self-DM was the newest outbox row. The 12 newest outbox rows and the 3 newest own turns held no fact the handover lacked. Steps 2 to 4 rebuild what a compaction lost; after a sunset the handover carries it by contract (`session-sunset` Step 8).
- **Step 6 loads loader diagnostics on every run.** About 2 KB of the payload (the Kimi and OpenCode load-proof and repair routes) is needed only when the proof is absent.

## The Architectural Reality

- `agents-md/sections/1500-mailbox-check-protocol.md` is the source of `§mailbox_check_protocol` in each consumer's generated `AGENTS.md`. That file is turn-loaded and stands at 23,285 B in `neomjs/neo`.
- `.agents/skills/context-recovery/references/context-recovery-workflow.md` §3, steps 1, 1a, 2, 3 and 6; 8,610 B at `origin/dev` (1.3.1).
- `.agents/skills/ticket-create/references/ticket-create-workflow.md` §1a (ii).
- A `list_messages` row carries no body in any projection (`neo-agent-brain#1254`), so `get_message` on a direct id is needed either way. `counts` marks no row as seen (its AC-4), so read state does not change with the cheaper call.
- Design authority: "The recovery runbook's mailbox step becomes: `counts` first, then `get_message` on the direct ids, then `compact` on the broadcast page only when the counts show something unusual" (neomjs/neo-agent-brain#1254, The Fix, item 2).

## The Fix

One PR in this repository. Section 1500 is turn-loaded, so `turn-memory-pre-flight` and `create-skill` run first.

1. Section 1500: `list_messages({status:'unread', projection:'counts'})`. The rest of the sentence stays.
2. Recovery step 1: `counts` first, then `get_message` on each entry of `unreadDirectMessageIds`, then a `compact` page only when the counts show an unusual class or sender.
3. Recovery step 1a, the outbox: `projection: 'compact', limit: 10`, newest first; page on only while rows are newer than the handover's `sentAt`.
4. Recovery step 3: `detail: 'summary', nResults: 5`; re-query one hit with `detail: 'full'` when its summary is not enough. The paragraph "Use `detail: 'full'` only when…" moves next to the call it governs.
5. A currency rule ahead of step 2: when step 1a finds a handover and no own outbox row is newer than it, step 2 runs with `limit: 3`, and steps 3 and 4 are skipped unless one of those three turns names something the handover lacks. Without a handover, or after a compaction, the steps run as today. Falsifier: a recovery on the short path that later re-derives a fact the long path would have surfaced; the specimen goes on this ticket.
6. Step 6 keeps the quarantine rule and one line per harness that names its proof. The diagnosis routes move to a reference file, read when the proof is absent.
7. `ticket-create` §1a (ii): add `projection: 'compact'`.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `§mailbox_check_protocol` (section 1500) | `neo-agent-brain#1254` item 2 | the per-turn check names `projection: 'counts'` | a plane older than `#1254` rejects the parameter; the same sentence already routes an erroring check to `/self-repair` | the section itself | the generated `AGENTS.md` diff in the PR |
| `context-recovery-workflow.md` §3 | `neo-agent-brain#1254`, `#1280` | steps 1, 1a and 3 name their parameters; the currency rule; step 6 by reference | no handover, or a compaction: the long path | the payload | the size table below |
| `ticket-create-workflow.md` §1a (ii) | the sweep's own text ("subject-scan stays primary") | `projection: 'compact'` | none | the payload | the PR diff |

Decision Record impact: none.

## Acceptance Criteria

- [ ] The four call sites carry the parameters of the table. The PR body lists every remaining `list_messages` and `query_raw_memories` call under `agents-md/` and `.agents/skills/` with the reason it stays full-size.
- [ ] The summary-first rule sits beside the step 3 call.
- [ ] The currency rule states its condition, its fallback and its falsifier.
- [ ] `context-recovery-workflow.md` is smaller than 8,610 B; section 1500 grows by at most 30 B; `check-substrate-size` passes.
- [ ] The PR body carries one recovery run on the new text from a seat with a 258k window: the answer size of each call, and the session gauge before and after.
- [ ] `standalone-context-recovery.md` is unchanged; it calls no Memory Core tool.

## Out of Scope

- The size of the `counts` answer: neomjs/neo-agent-brain#1555.
- A broadcast drain at sunset (`neo-agent-brain#1254`, item 3). With bounded reads the unread backlog costs nothing to carry; the backlog itself is neomjs/neo-agent-brain#87.
- Consumer pin bumps and regenerated `AGENTS.md` files: the usual release path. The per-turn line reaches a seat with its next pin.
- The session floor before the first message (tool schemas, skills listing, memory files): harness and seat configuration.

## Avoided Traps

- Changing the tools' defaults: `#1254` AC-3 and `#1280` kept `full` so existing callers and specs stay byte-identical. The rules name the cheap form instead.
- A drain with `includeUnseen: true` to shorten the unread list: it marks read what no seat saw, which `withheldUnseenCount` exists to prevent.
- Removing steps 2 to 4: they are the recovery after a compaction. Only the handover case shortens them.

## Related

neomjs/neo-agent-brain#1254, neomjs/neo-agent-brain#1280, neomjs/neo-agent-brain#1290, neomjs/neo-agent-brain#1555, neomjs/neo-agent-brain#87; #221 (the same rule in `memory-mining`), #312 (the last change to this runbook), #289 (sunset Step 1).

handoff: @neo-gemini-maryam, first refusal. She changed this runbook in #312, and a 258k window is where the receipt in the fifth criterion means most. No `good first issue` label: the leaf has a named owner and changes turn-loaded text.

Live latest-open + closed sweep: latest 20 open, created-descending, and closed searches (mailbox unread projection; recovery, sunset, token, budget) at 2026-11-09T23:32Z; no equivalent or prior exclusion found. 30 open (orange): no owning ticket or epic for the read sizes of the rules. A2A in-flight claim sweep: the newest rows of `list_messages({status: 'all'})` at 23:33Z, no claim on this scope. MC sweep: "boot mailbox check costs a page of unread broadcast rows every turn, recovery reads too many tokens", 5 results, no prior decision found (two seats drained their backlogs by hand in July). Own-assignment sweep: 1 open here (#191), not overlapping. Structure map (Brain `4605d893`): every touched file exists; the one new reference file sits in `context-recovery/references/` beside `standalone-context-recovery.md`.

Body history: a fifth row (the structure map) was added at 2026-11-09T23:45Z, three minutes after the claim, and moved to its own leaf #329 at 2026-11-10T00:00Z. The criteria above are the ones claimed.

Origin Session ID: 6bc6e94b-0b0c-4f41-ad1f-ec6dcdbfdf6d

Retrieval Hint: "recovery read sizes counts projection summary detail handover currency rule"

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session 6bc6e94b-0b0c-4f41-ad1f-ec6dcdbfdf6d


## Timeline

- 2026-11-09T23:36:30Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-09T23:36:30Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-09T23:36:30Z @neo-fable-mnemosyne added the `performance` label
- 2026-11-09T23:37:17Z @neo-fable-mnemosyne cross-referenced by #1254
- 2026-11-09T23:37:21Z @neo-fable-mnemosyne cross-referenced by #87
- 2026-11-09T23:42:37Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-09T23:52:53Z @neo-gemini-maryam cross-referenced by PR #328
- 2026-11-09T23:59:53Z @neo-fable-mnemosyne cross-referenced by #329
- 2026-11-10T00:00:02Z @neo-fable-mnemosyne cross-referenced by #1556
- 2026-11-10T00:12:02Z @neo-gemini-maryam referenced in commit `20f7009` - "fix(skills): address review feedback on recovery outbox bound and load proof (#326)"
- 2026-11-10T00:29:47Z @tobiu referenced in commit `4310a5f` - "perf(skills): use cheap mailbox and memory projections in loaded rules (#326) (#328)

* perf(skills): use cheap mailbox and memory projections in loaded rules (#326)

* fix(skills): address review feedback on recovery outbox bound and load proof (#326)"
- 2026-11-10T00:29:47Z @tobiu closed this issue
### @neo-fable-mnemosyne - 2026-11-10T01:21:45Z

**First sunset restart on the short path: Tue 2026-11-10, 1M window (Claude Code)**

Run from the v1.3.2 text in the Skills clone; the installed runbook is still 1.2.0 (last point).

| read after | session gauge | of which messages |
|---|---|---|
| the runbook, `counts`, 7 direct bodies (the handover and its addendum among them) | 111,370 | 40,895 |
| + the map's first block, the plate, the pin check | 116,666 | 46,191 |

The floor before the first message is 69,513 (system prompt, tools, memory files, skill list), so no reading exists below it. Last night's long-path run on this seat ended at 148.7k. The short path ends 32k lower.

The two conditional steps:

- **Recency, `limit: 3`:** three summaries, each already in the handover. Steps 3 and 4 skipped.
- **Outbox, `compact`, `limit: 3`:** one own row was newer than the handover, a receipt I sent after the sunset (01:11:59Z against 01:09:20Z). By the letter the currency condition fails there and the long path is due. I read that row in full (0.9 KB) and stayed short. Specimen for the rule text: a newer row that step 1a reads in full should not void the rule; only one left unread should.

**The fix is not loaded yet.** `neomjs/neo` pins `neo-agent-skills` 1.2.0, npm has 1.3.2, and no bump PR is open (Dependabot moved this pin before). A seat rooted in the engine checkout still reads the old runbook.

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session fda09208-5d6a-4fc7-a0d4-69a53fdf2dec


