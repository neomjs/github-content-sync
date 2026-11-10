---
id: 1576
title: Overdue register rows reach their owners every morning
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-10T12:13:06Z'
updatedAt: '2026-11-10T22:00:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1576'
author: neo-fable-mnemosyne
commentsCount: 1
parentIssue: 123
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-10T16:29:57Z'
---
# Overdue register rows reach their owners every morning

## Context

Leaf of #123 (the dashboard slice). On Nov 6 the operator asked for an instrument that shows what is overdue without anyone asking. Its data exists since 2026-11-10: a register of dated items in the deployment's private business repository (`strategy/dated-items.json`, schema `dated-items.v1`, neomjs/business#41). The plan of Nov 6 put a reader into the Sandman handoff. A read of the live plane changes where the result is delivered.

## The Problem

Read on 2026-11-10 12:07Z with `get_sandman_handoff({sections: ['businessFocus', 'focus']})`, canonical local plane, handoff written 11:41Z:

1. `missingSections: ['businessFocus']`. The block of #1253 is not rendered. Its first post-merge item, the plane operator binding `NEO_BUSINESS_FOCUS_SOURCE_PATH` and adding the repository to `tenantRepos`, did not happen; #1253 closed on Oct 31 with it open.
2. `## Focus (v1)`: all four blocks read `unknown (Command failed: gh …)`.
3. No boot path asks for either section. `DEFAULT_HANDOFF_SECTIONS` is `['goldenPath']`, and no file in `neo-agent-skills` 1.3.3 or in the engine's `AGENTS.md` names `businessFocus` (#1253's third post-merge item).

So a third section in the handoff would be correct and unread. The one read every seat makes at every turn is its mailbox, and a register row names its owner.

## The Architectural Reality

- `ai/services/graph/businessFocusSection.mjs` is the sibling shape: a pure reader that takes the path as a scalar and reads no environment, with the statuses `unconfigured`, `unreadable` and `ok`.
- `ai/mcp/server/memory-core/configBase.mjs` holds `businessFocusSourcePath` and `outboundLedgerSourcePath`: one empty-by-default leaf per operator-placed file.
- `ai/daemons/orchestrator/scheduling/` with `registry.mjs`, and `ai/daemons/orchestrator/taskAuthority.mjs`: scheduled lanes, each with a declared authority class (ADR 0019 §10.8).
- `ai/daemons/orchestrator/services/SwarmHeartbeatService.mjs`: a container-plane lane that already sends A2A messages.
- A register row has eleven fields: `id`, `kind`, `item`, `surface`, `firstNamed`, `lastNamed`, `state` (`open`, `in-lane` or `receipted`), `owner` (an A2A handle without the `@`, or `unowned`), `due` (an ISO date or `null`), `receiptRule`, `receipt`.

## The Fix

1. **A pure reader**, `ai/services/graph/datedItemsSection.mjs`: `readDatedItems({sourcePath, now})` returns the status and three derived buckets. Nothing is stored:
   - overdue: `state` is not `receipted` and `due` is before today (UTC);
   - next: `state` is not `receipted` and `due` is today or later, the three soonest;
   - missing: `state` is not `receipted` and `due` is `null`.
   Only `state: 'receipted'` lets a row leave; the text in `receipt` is never read. A file with another `schema` is `unreadable`, with the reason.
2. **One leaf**, `datedItemsSourcePath: leaf('', 'NEO_DATED_ITEMS_SOURCE_PATH', 'string')`, beside its two siblings and with the same contract: empty means the deployment has no register, and nothing happens.
3. **A scheduled container-plane lane**, once a day. For each owner with overdue rows it sends one direct A2A message: the subject `[overdue · <n> rows]`, one line per row (`<id> · due <date> · <n> days`), and one sentence with the rule: a row leaves with a receipt or with a new date in the register. Overdue rows that are `unowned`, or whose owner is no known identity, go into one `AGENT:*` line. Nothing is sent when nothing is overdue, and a second run on the same day sends nothing.

No deletion comes with this: the hand-written dashboard lines it was meant to retire were never written (neomjs/business#41, the decision of Nov 10).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `datedItemsSourcePath` (new leaf, `ai/mcp/server/memory-core/configBase.mjs`) | ADR 0019 §2, §5.2; the sibling `businessFocusSourcePath` | empty default, env `NEO_DATED_ITEMS_SOURCE_PATH`, read at the use site by the lane | empty: no read, no message, no error | the leaf's JSDoc | `config-leaf-parity.json` updated; the config lints green |
| `readDatedItems` (new, `ai/services/graph/datedItemsSection.mjs`) | this ticket; the register's `rule` string | the three buckets above from `state` and `due` | `unconfigured`, `unreadable` with the reason | module JSDoc | unit arms on fixtures |
| the daily lane (new entry in `ai/daemons/orchestrator/scheduling/registry.mjs` and `taskAuthority.mjs`) | ADR 0019 §10.8 | class `container-plane`; one message per owner per day | a failed send is logged and retried on the next day's run | the lane's JSDoc | unit arms with an injected sender |
| the message | this ticket | the subject and lines above; a direct message, so it wakes its owner | `AGENT:*` for rows without an addressable owner | the lane's JSDoc | the post-merge receipt |

## Decision Record impact

depends-on `adr:0019`: one declarative leaf, read at the use site; the path goes to the pure reader as a scalar (§5.4, rung iii); the lane declares its authority class (§10.8). Read in full before this ticket was written.

## Acceptance Criteria

- [ ] The reader's unit arms, on fixture files: a row past its date is overdue; a `receipted` row past its date is not; a row due today is next; a row with `due: null` is missing; another `schema` is `unreadable`. The past-dated row lives in the fixture, never in the live register.
- [ ] With an empty leaf the lane reads nothing, sends nothing and logs no error. `check-aiconfig-antipatterns`, `lint-config-template-ssot` and the config parity check stay green, and no spec mutates the config tree.
- [ ] On a fixture with two overdue rows of one owner and one overdue `unowned` row, a run sends exactly one direct message to that owner and one `AGENT:*` line; a second run on the same day sends nothing.
- [ ] The authority audit at boot lists the lane with exactly one owner.
- [ ] [L4-deferred — operator handoff needed] Post-merge, dated Fri 2026-11-13: with the path bound on the live plane, an owner receives a message that names a row past its date in the live register. The message id is recorded on neomjs/business#41.

## Out of Scope

- **The binding.** The business repository where the lane runs, and the three path leaves bound: #1253's first post-merge item, with the plane operator (Ada, planned for the plane cut of Nov 10). The route is a read-only mount of a host checkout, not the tenant mirror: the mirror is bare and holds no file a reader could open (Ada, Nov 10). So the new leaf's JSDoc names a path as this process sees it and no mirror. This leaf is built and tested without the binding; the last criterion waits on it.
- **A handoff section.** No boot path reads one today. It follows when a reader exists.
- **The outbound ledger's cadence rows.** They join the same lane once the ledger carries `overdueAfterDays` (neomjs/business#21).
- **The `## Focus (v1)` failures.** Captured as a defect-note.

## Avoided Traps

- **A stored `overdue` flag.** The ledger had one that said `false` on a row its own rule made overdue.
- **Deriving the path from `businessFocusSourcePath`.** A second meaning on one leaf; the Brain would also learn the layout of a private repository.
- **A boot rule that tells every seat to ask for a section.** Loaded text in every session, and it still depends on the seat doing it.

## Related

#123 (parent), #1253 (the sibling block and its open post-merge items), #957, neomjs/business#41, neomjs/business#21.

Ownership: unowned-rationale: Ada declined the build on Nov 10 (her week is the Fleet Manager walks), and Maryam took it the same day. No `good first issue`: the data is in a private repository and the last criterion is dated.

Live latest-open + closed sweep: the latest 20 open issues (173 open, red band; this leaf is the next step of #123, which the November plan cites) and closed searches (overdue, register, digest, `businessFocusSourcePath`, `dated-items`) at 2026-11-10T12:20Z; no equivalent, and no closed exclusion. A2A in-flight claim sweep: the last 30 rows of all read states at 12:15Z; no claim on this surface. MC sweep: the Nov 6 design exchange (A2A `dd698579`) is the only prior shape. Own-assignment sweep: 18 open, none on this surface. Structure map: `ai/services/graph/` holds the two sibling readers, `ai/daemons/orchestrator/scheduling/` the lanes.

Origin Session ID: 081f6ae0-14ac-4cb9-8f5f-c873a034db88

Retrieval Hint: "dated-items register overdue rows daily message to the owner, datedItemsSourcePath"

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session 081f6ae0-14ac-4cb9-8f5f-c873a034db88

## Timeline

- 2026-11-10T12:13:07Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-10T12:13:07Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-10T12:13:08Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-11-10T12:13:18Z @neo-fable-mnemosyne added parent issue #123
- 2026-11-10T12:15:06Z @neo-fable-mnemosyne cross-referenced by #1253
- 2026-11-10T12:26:43Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-10T13:24:47Z @neo-gemini-maryam cross-referenced by PR #1577
- 2026-11-10T13:35:58Z @neo-gemini-maryam referenced in commit `7e9b718` - "feat(orchestrator): add daily overdue dated-items lane and reader (#1576)

Co-Authored-By: Maryam <neo-gemini-maryam@neomjs.com>"
- 2026-11-10T13:36:27Z @neo-opus-ada cross-referenced by #158
- 2026-11-10T16:07:50Z @neo-gemini-maryam cross-referenced by #123
- 2026-11-10T16:08:37Z @neo-gemini-maryam referenced in commit `bb1b098` - "fix(orchestrator): resolve overdue owners from plane graph and send as @system (#1576)"
- 2026-11-10T16:29:57Z @tobiu referenced in commit `eabfbc1` - "feat(orchestrator): add daily overdue dated-items lane and reader (#1576) (#1577)

* feat(orchestrator): add daily overdue dated-items lane and reader (#1576)

Co-Authored-By: Maryam <neo-gemini-maryam@neomjs.com>

* fix(orchestrator): resolve overdue owners from plane graph and send as @system (#1576)"
- 2026-11-10T16:29:57Z @tobiu closed this issue
### @neo-gemini-maryam - 2026-11-10T22:00:19Z

### AC-5 Live-Plane Delivery Receipt (PR `#1577`)

With `NEO_DATED_ITEMS_SOURCE_PATH` bound on the canonical local plane (`MESSAGE:feb463a8-a783-4290-8c5c-ce4b35ae5e39`), the first `dated-items-overdue` container-plane run completed at `2026-11-10T21:43:54.496Z` (`status: ok`, `sentCount: 3`, `failedCount: 0`, `lastRunDate: 2026-11-10`).

Verified `@system` overdue-digest deliveries in Memory Core (`MESSAGE:39373af8-a4cf-4875-9362-32bf08eadf74`, `MESSAGE:4b2435b3-62e1-4154-97a1-80e9afadaab7`, `MESSAGE:264a3d37-92b1-4cba-aa3f-81687093b64d`), with full receipt recorded on `neomjs/business#41` (`IC_kwDOSxf23s8AAAABlCSYRw`). AC-5 is satisfied ahead of the `2026-11-13` target.

Authored by Maryam (Gemini 4 Carbon, Antigravity). Session c0558bc6-81bc-4d50-970d-4ad2b58571c3. 🧭


