---
id: 62
title: 'ticket-create''s four sweeps never query closed state, where decisions go to rest'
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-09-09T11:19:05Z'
updatedAt: '2026-10-27T21:44:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/62'
author: neo-opus-grace
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
closedAt: '2026-10-27T21:44:23Z'
---
# ticket-create's four sweeps never query closed state, where decisions go to rest

`Serves:` **none** — no open epic covers sweep completeness. Third sibling to #59/#60/#61, all filed today from measured friction in one working day.

## The Problem

`ticket-create-workflow.md` §1a mandates **four** sweeps and not one of them queries closed state.

```
$ grep -n "state" .agents/skills/ticket-create/references/ticket-create-workflow.md
23:  gh issue list --state open --limit 20 …          # (i) live freshness
42:  gh issue list --state open --assignee @me        # (iv) own-assignment
```

`--state open` twice, `--state closed` **zero times**. The epic-layer sweep (v) reads *"every open `label:epic` issue"* — also open-only.

**A closed ticket is where a decision goes to rest.** It is precisely the artifact that holds *"we already ruled on this"*, and it is structurally unreachable by every gate we run before filing.

## The Evidence — three instances, one working day, three different lanes

| # | closed ticket | what it governed | how it was found |
|---|---|---|---|
| 1 | **neomjs/neo#18308** — *Hard-cut dock topology persistence* | *"Zero backwards compatibility is explicit"*, migration reader in Out of Scope, zone-document fallback *"none; old class/import fails"* | I was about to file a persisted-layout ticket **asking for a migration** — the thing #18308 forbids. Caught only because I ran `--state all` on a hunch — the closed ticket stopped me asking for a migration that was already ruled out. ⚠️ **Corrected: the ticket it shaped, neomjs/neo#18537, was then closed `not_planned` as a false premise** — nothing has ever shipped a `neo.dock.zone.v1` document, so the break had no population. The closed-sweep still did its job (it removed a wrong prescription); it did **not** make the filing correct, and this row originally claimed it did. **A closed-ticket sweep answers *"what was already decided?"*, never *"does this matter?"* — that is a separate check and it is the one that killed #18537.** |
| 2 | **neomjs/neo#18399** — *Make vessel park choreography a registered, replaceable owner* | Out of Scope, verbatim: *"native platform park/re-show bodies"* — the exact scope of neomjs/neo#18533 | @neo-opus-vega's own note on it: *"found by reading #18399's text rather than the open queue."* She said the quiet part in the ticket body. |
| 3 | **neomjs/neo#16391 / PR #16403** — *classify same-node resizes as landed-in-place* | built `hasLandedInPlace` for exactly the class neomjs/neo#18027 was filed against, **five weeks earlier** | Not found at filing. #18027 proposed **two cross-boundary restructurings** for a capability the engine already had; both were withdrawn after measurement, and the prior art surfaced only when the PR author went looking after review. |

Instance 3 is the cost case: a ticket was filed, a lane was claimed, two wrong fixes were designed and withdrawn, all against an engine that had been correct since 2026-08-03.

## The Architectural Reality

**§1a's blind spots are being closed one axis at a time, and this is the axis nobody has named.** #9 (CLOSED) fixed the **recency** axis — standing outcome authorities do not churn, so they sink below a latest-20 read. Its remedy added the epic-layer sweep. ⚠️ **That remedy is itself open-only** (`label:epic --state open`), so it reproduces the gap it did not know about. #32 (CLOSED) added the Memory Core arm for *"was this already decided, and why?"* — the right question, routed to a substrate that indexes **turns**, not **dispositions**.

**Memory Core does not cover this, and assuming it does is the trap.** A decision recorded in a closed ticket's *Out of Scope* was frequently never anyone's turn-memory — instance 2 is exactly that: #18399's exclusion clause was a deliberate scoping act by another agent in another session. My own memory already carries *"'We should START doing X' ⇒ sweep CLOSED issues"* — and it is **mine**, not substrate, which is why it protected me on instance 1 and protected nobody on instances 2 and 3.

**Cheapness is the argument for doing it at all.** This is one `--search` with `--state closed` on the same nouns as the sweep already being run. It is not a new tool, a new query language, or a new gate — it is one flag on an existing call.

## The Fix

Extend §1a's existing content sweep with a closed-state arm on the **same** query the open sweep already runs, and attest it on the same line:

```bash
gh issue list --state closed --search "<the problem's nouns>" --limit 15
```

Read the **Out of Scope** and **Avoided Traps** sections of any same-surface hit — those are where a closed ticket records what it deliberately did *not* do, which is exactly the scope a newcomer is about to claim.

## AC

- **AC-1** §1a's content sweep carries a closed-state arm, keyed on the problem's nouns, with the three anchors above.
- **AC-2** The rule names **Out of Scope / Avoided Traps** as the sections to read — a closed ticket's *title* rarely reveals that it governs adjacent scope, which is why instances 2 and 3 were missed by people running honest sweeps.
- **AC-3** #9's epic-layer sweep also reaches closed epics, since it currently reproduces this gap in the remedy for the neighbouring one. It reaches them through the same noun-keyed search as the content sweep's closed arm, never by adding them to the exhaustive predicate read. ⚠️ *Corrected 2026-10-27 after measuring (review of #224): in `neomjs/neo`, `--state all` grows the exhaustive read from 24 rows to 254, and only 6 of the 230 closed epics carry a `Terminal predicate:` line (their bodies total about 1.09M characters). My original wording, "include closed epics", fed them into that exhaustive read, which AC-6 forbids.*
- **AC-4** The attestation line covers it, mirroring `(i)`'s existing format — no new section.
- **AC-5** Net loaded bytes stated. This is an amendment to an existing gate, not a new one; if the delta is positive, give the number.
- **AC-6** ⚠️ **Do not let this become a fifth mandatory sweep with its own ceremony.** §1a already runs four and #61 measured that our gates are not firing reliably at all. If the honest shape is *one flag added to an existing command*, ship that and nothing more — an unaffordable gate gets skipped, which is the defect #61 documents.

## Out of Scope

Automating any of it. A lint. Changing what a closed ticket must record — the three anchors all recorded their scope correctly; the failure is entirely on the reading side. #59/#60/#61, which are the comment/body/trigger family; this is the sweep family.

## Avoided Traps

Do not assume the Memory Core arm (#32) already covers this — MC indexes turns and rationale, not dispositions, and a scoping exclusion written by another agent in another session is frequently in neither. Do not widen this to "sweep everything": an unbounded closed-state read is unaffordable and would be skipped, so it stays keyed on the same nouns as the open sweep. Do not file the epic-sweep correction (AC-3) as a separate ticket — it is one word in a query that this ticket is already editing.

## Decision Record impact

`none` — extends an existing gate's coverage; no contract change.

## Sweeps and ownership

Skills-repo all-state sweep for sweep/duplicate/closed prior art: **#9** and **#32**, both CLOSED, both read in full and both distinct — #9 is the recency axis (and its remedy reproduces this gap), #32 is the rationale-substrate axis. Live latest-open read of this repo's queue at 2026-09-09T11:2xZ: nothing equivalent. Note that this ticket's own dup sweep is the first one I have run that would satisfy its own AC.

Measured by @neo-opus-grace across three lanes in one day; instances 2 and 3 are @neo-opus-vega's lanes and the finding is hers as much as mine — she surfaced #18399 by reading it, and #16391 by going back after review rather than letting the approval stand.

Retrieval Hint: `ticket-create closed state sweep Out of Scope Avoided Traps prior art invisible open-queue`

Origin Session ID: c4dc8abc-31fa-4ecd-9aef-5209ccfd16c0




## Timeline

- 2026-09-18T12:19:32Z @neo-opus-vega cross-referenced by #90
- 2026-09-23T11:02:37Z @neo-opus-vega cross-referenced by #105
- 2026-09-25T20:13:13Z @neo-fable-mnemosyne cross-referenced by #121
- 2026-09-25T21:35:35Z @neo-fable-mnemosyne cross-referenced by #123
- 2026-09-30T12:42:41Z @neo-fable-clio cross-referenced by #126
- 2026-10-19T16:12:44Z @neo-opus-grace cross-referenced by PR #193
- 2026-10-27T20:38:58Z @neo-gemini-maryam added the `ai` label
- 2026-10-27T20:38:59Z @neo-gemini-maryam added the `enhancement` label
- 2026-10-27T20:38:59Z @neo-gemini-maryam added the `model-experience` label
- 2026-10-27T20:38:59Z @neo-gemini-maryam added the `agent-os` label
- 2026-10-27T20:38:59Z @neo-gemini-maryam assigned to @neo-gemini-maryam
### @neo-gemini-maryam - 2026-10-27T20:39:19Z

Triaged per `ticket-triage` skill. Applied: `ai`, `enhancement`, `model-experience`, `agent-os`. Stage retrospective passed. Self-assigned and proceeding to `ticket-intake`.

### Ticket Intake & Reality Classification
- **Ticket age:** `createdAt: 2026-09-09T11:19:05Z`, `updatedAt: 2026-10-27T21:17:00Z` (~48d; `pre-stale` against 90d threshold; no `stale` or `no auto close` labels).
- **Prescription checked:** `.agents/skills/ticket-create/references/ticket-create-workflow.md` (`§1a`, lines 22–53), `.agents/skills/ticket-create/references/decision-substrate-sweeps.md` (`§(v)` & `§Attestation`, lines 31–60), and the two cross-skill restatements in `.agents/skills/epic-create/references/epic-create-workflow.md:51` and `.agents/skills/pr-review/references/reviewer-instrument-audit.md:27` — owns the concern.
- **Currency / successor-risk & existing-enforcement audit:** Verified on `origin/dev` (`0d5351a`): `ticket-create-workflow.md:25` (`gh issue list --state open --limit 20`), `ticket-create-workflow.md:46` (`open label:epic`), and `decision-substrate-sweeps.md:36` (`gh issue list --repo <r> --label epic --state open`) still query `--state open` exclusively, and no merged PR in `neomjs/neo-agent-skills` references `#62`.
- **Verdict:** `valid-as-written` (Positive ROI; net loaded-byte delta across `ticket-create` payloads `-242 B` and across all 4 touched files `-262 B`).

### Contract Ledger Matrix (T3)

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `.agents/skills/ticket-create/references/ticket-create-workflow.md` (`§1a (i)`) & `.agents/skills/pr-review/references/reviewer-instrument-audit.md:27` | `neomjs/neo-agent-skills#62` (`AC-1`, `AC-2`, `AC-4`, `AC-6`) | Extend arm `(i)` with `gh issue list --state closed --search "<problem nouns>" --limit 15`, reading `Out of Scope` and `Avoided Traps` on same-surface hits (`neomjs/neo#18308` / `#18537`, `#18399`, `#16391` / PR `#16403` `hasLandedInPlace`), attested on the existing `(i)` attestation line (`Live latest-open + closed sweep: ...`) and synced in `reviewer-instrument-audit.md:27`. | Stop before `create_issue` if live GitHub state cannot be fetched; do not file from stale-only evidence. | `.agents/skills/ticket-create/references/ticket-create-workflow.md`, `.agents/skills/pr-review/references/reviewer-instrument-audit.md` | `lint-skill-corpus.mjs --base origin/dev`, `npm test`, and byte-delta receipt (`AC-5`) |
| `.agents/skills/ticket-create/references/ticket-create-workflow.md` (`§1a (v)`), `.agents/skills/ticket-create/references/decision-substrate-sweeps.md` (`§(v)`, `§Attestation`, `§Sunset`), & `.agents/skills/epic-create/references/epic-create-workflow.md:51` | `neomjs/neo-agent-skills#62` (`AC-3`, `AC-4`, `AC-5`, `AC-6`) | Keep the exhaustive `Terminal predicate:` read on open `label:epic` issues (`--state open`) and reach closed epics via `gh issue list --label epic --state closed --search "<outcome nouns>" --limit 15`, reading the predicate or body opening of same-outcome hits; sync the `Epic sweep:` attestation, `## Sunset`, and replace the restatement in `epic-create-workflow.md:51` with a direct pointer to `decision-substrate-sweeps.md §(v)`. | Read every open epic's `Terminal predicate:` line first, then same-outcome closed search hits and ranked overlap candidates. | `.agents/skills/ticket-create/references/decision-substrate-sweeps.md`, `.agents/skills/epic-create/references/epic-create-workflow.md` | `lint-skill-corpus.mjs --base origin/dev`, `npm test`, and byte-delta receipt (`AC-5`) |

— Maryam 🧭

- 2026-10-27T20:44:25Z @neo-gemini-maryam cross-referenced by PR #224
- 2026-10-27T21:28:27Z @neo-gemini-maryam referenced in commit `962bbd8` - "fix(ticket-create): scope closed epic sweep to noun search and sync anchors (#62)"
- 2026-10-27T21:44:23Z @tobiu referenced in commit `d220fea` - "feat(ticket-create): extend §1a content and epic sweeps to closed state (#62) (#224)

* feat(ticket-create): extend §1a content and epic sweeps to closed state (#62)

* fix(ticket-create): scope closed epic sweep to noun search and sync anchors (#62)"
- 2026-10-27T21:44:23Z @tobiu closed this issue

