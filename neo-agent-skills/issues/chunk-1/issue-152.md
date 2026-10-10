---
id: 152
title: An /overview skill answers where v1 stands and which views need love
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-fable-clio
createdAt: '2026-10-09T21:59:59Z'
updatedAt: '2026-10-10T00:18:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/152'
author: neo-fable-clio
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
closedAt: '2026-10-10T00:18:56Z'
---
# An /overview skill answers where v1 stands and which views need love

## Context

Graduated from [D#19493](https://github.com/neomjs/neo/discussions/19493) OQ-8 (viii): *the skill is partial by design — it reads the same section through the handoff tool and reports what the section carries; a handoff-only skill never invents an absent section; before the producer leaf lands it prints the rows' `Row state:` lines from GitHub and names the slice as absent; shipped with its package, pin and recipient-load witness.* The operator's goal of 2026-10-09: a peer in a fresh session answers *where are we for v1, what is missing* and *which views still need love* in one read. The producer half is neomjs/neo-agent-brain#957 (the `## Focus (v1)` section of the Sandman handoff); the receipts' line is #150's (`view: <key> · needs love: …` on neomjs/neo-agent-institution#505).

## The Problem

Today the two questions cost a session its first twenty minutes: the five rows' `Row state:` lines live on five Institution epics, the module cut line in the Discussion, the views' state in #505's comments, and the morning surface (`get_sandman_handoff`) carries none of it. Every peer re-derives the picture, or asks the operator. When #957 lands, the picture exists as a section — but nothing loads it on demand, and until then nothing answers at all.

## The Architectural Reality

- Skills are routers + payloads under `.agents/skills/<name>/` (ADR 0008); the manifest row, budgets and growth cap apply as on #150.
- `get_sandman_handoff` (Memory Core) returns the handoff with freshness metadata (`staleAfterMs`); a level-two section `## Focus (v1)` is #957's producer-owned unit, versioned by its `focus.v1` marker line and carrying four blocks (v1 rows, views, reach + outbound, what waits for the operator's word), each with its observation time and an `unknown` shape.
- The rows: the Institution `ROADMAP.md` table names rows 1–5 (`:27–31`) and links each epic; every epic carries one `Row state:` line (verified 2026-10-09 21:5xZ on #351 `ready`, #477 `unknown`, #312 `unknown`, #414 `failed`, #424 `ready`, each with its date and candidate). The words are steward-owned; `ready` is not `passed`.
- The views: #505's receipt comments carry the `view: <key> · needs love: <line>` line once #150's sweeps run; the keys are the stable view keys (#150 §2).
- A seat has GitHub (`gh` with the seat's credential); the skill reads GitHub from the seat (the host edge), never through the cloud producer — Euclid's clause on D#19493 holds for the Brain side; the skill side is a local read.

## The Fix

New skill `overview` (invoked `/overview`), payload-only:

1. `SKILL.md` — router ≤ 12 lines: triggers = a fresh session's first planning question; before an FM lane claim; the weekly planning slot; an operator's "where are we".
2. `references/overview-read.md` — the read order and the one-screen shape, in #957's four blocks so the skill and the section agree:
   - **(1) the section first:** `get_sandman_handoff`; if `## Focus (v1)` is present, print it with its freshness and stop at the blocks it carries; a stale envelope is printed as stale, never silently.
   - **(2) the fallback, block by block, when the section or a block is absent:** *v1* — read the ROADMAP's row table for the five epic numbers (never hardcoded), then each epic's `Row state:` line verbatim with its date; *views* — the `needs love:` lines from #505's receipt comments with their binding (build class, profile, time); the governing line per key is the newest of the highest evidence class (an installed line is replaced only by a newer installed line, a source line never resolves it); `none observed` is coverage, not a finding; keys without a line are `unassessed`; *what waits for the operator's word* — open PRs with an approving review, discovered by search and then read per candidate through the source-owned merge-readiness projection (`get_conversation` with `projection: 'merge-readiness'`, the Brain's `validateMergeReady`): `ready for the operator` only on the projection's ready verdict, `approved · not ready: <its reasons>` otherwise, `approved · readiness unverified` when the projection is unavailable — the skill computes no readiness of its own; non-PR decisions named as outside the read's coverage; *reach + outbound* — `unknown` with the reason (no seat-side source until #957's host-edge reader or D#19500's analytics).
   - **(3) the shape:** one screen; every block with its as-of time and source link; the section's words and the rows' words copied raw; the questions answered in two lines at the top (*v1: n of 5 rows passed · missing: …* / *views needing love: k*).
3. `assets/overview-screen.md` — the one-screen template.
4. The manifest row; the version bump.

Decay clause: when the section carries all four blocks (the `focus.v1` marker with four block headings), step (2) retires to a pointer and the skill is a one-call read; the retirement trigger is the first `/overview` print whose every block came from the section. Recorded in the payload head.

Decision Record impact: aligned-with ADR 0008; the Discussion carries no ADR classification (`Not needed`).

## Discussion Criteria Mapping

| D#19493 OQ-8 (viii) criterion | AC here |
|---|---|
| reads the same section through the handoff tool and reports what it carries | AC-2 |
| never invents an absent section; names it absent | AC-2 |
| before the producer lands: the rows' `Row state:` lines from GitHub | AC-3 |
| shipped with its package, pin and recipient-load witness | AC-5 |

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `.agents/skills/overview/SKILL.md` | ADR 0008 | ≤ 12-line router, triggers in `description` | — | itself | lint |
| `references/overview-read.md` | D#19493 OQ-8 (viii); #957's block contract | section first; per-block fallback from the seat's GitHub; raw words | an unreadable source → `unknown` + reason, never a guess | itself | the first print (AC-6) |
| the output shape | #957's four blocks + stable view keys + the design-sweep receipt's binding | one screen, as-of per block, two answer lines on top (`needing love` counts affirmative governing lines only; `unassessed` named), view lines with build class/profile/time, operator candidates with a readiness label | a block absent everywhere → `unknown`; a key without a line → `unassessed`; a candidate not readiness-read → labelled unverified | the template | the first print; the six synthetic cases in the payload (§3a) |
| `skills.manifest.json` row | the manifest schema | mirrors the frontmatter | — | README | lint |
| the consumer load | #140 / #144 pattern | the release reaches the pins; a non-author seat loads `/overview` fresh | — | PR body | the load receipt (AC-5) |

## Acceptance Criteria

- [ ] AC-1 Router ≤ 12 lines with frontmatter and the payload path; manifest row; `lint-skill-corpus` green.
- [ ] AC-2 The payload reads the section first through `get_sandman_handoff`, prints its freshness, and names an absent section or block as absent — never a fabricated block.
- [ ] AC-3 The fallback reads the five epic numbers from the ROADMAP table and prints each `Row state:` line verbatim with its date; `ready` prints as `ready`.
- [ ] AC-4 The PR body carries the load-effect audit and the commit the `[skill-growth-justified: …]` marker; the version bumps.
- [ ] AC-5 (post-merge) A non-author seat's fresh session loads `/overview` and posts the load receipt here.
- [ ] AC-6 (post-merge) The first print from a non-author seat answers the two questions in one read, with the section absent (today's state) — receipt on this ticket; when #957 lands, a second print with the section present.

## Out of Scope

The producer (neomjs/neo-agent-brain#957); the Fleet Manager pane (neomjs/neo-agent-institution#312's leaf, after v1); reach/outbound sources on the seat side (D#19500); any hand-kept ledger file.

## Avoided Traps

- Hardcoded epic numbers or row names: the ROADMAP table is the source.
- A second status authority: the rows' words are printed raw, never promoted or summarized into a verdict.
- A fabricated block: absent is printed as absent.
- Tool mechanics in the payload: `get_sandman_handoff`'s parameters live in its description.

## Related

D#19493 OQ-8; #150 (the receipts' line); neomjs/neo-agent-brain#957 (the producer); neomjs/neo-agent-institution#505, #312; Skills #140, #144.

Live latest-open sweep: the latest 20 open issues of this repository, created-descending, at 2026-10-09 21:5xZ (newest #150); no equivalent. A2A in-flight claim sweep at 21:4xZ (30 newest, all read-states): none on this scope. Memory Core rationale sweep: the 2026-10-09 OQ-8 origin turns (the skill-first, Brain-tool-after sequence); no prior decision. Own-assignment sweep: #150 is the sibling, not a duplicate. Structure map: N/A (this package; sibling precedent `.agents/skills/design-sweep/`). Meta-skill sweep: the authoring guide read on #150 the same evening.

unowned-rationale: sequenced after #150's merge (the version line serializes skills PRs); open to any seat that wants it before then.

Origin Session ID: cf93d406-6f17-4f10-9f72-9768482edfb1

Retrieval Hint: "overview skill where are we for v1 which views need love Row state lines Focus (v1) section fallback"



## Timeline

- 2026-10-09T22:00:01Z @neo-fable-clio added the `enhancement` label
- 2026-10-09T22:00:01Z @neo-fable-clio added the `ai` label
- 2026-10-09T22:00:02Z @neo-fable-clio added the `agent-os` label
- 2026-10-09T22:07:37Z @neo-opus-vega cross-referenced by #651
- 2026-10-09T23:03:36Z @neo-fable-clio assigned to @neo-fable-clio
### @neo-fable-clio - 2026-10-09T23:05:36Z

## First print (author, pre-merge — the fallback path, from the seat) · 2026-10-09 23:0xZ

The skill is authored on `clio/152-overview-skill` (810eddd, stacked on #151's branch; the PR opens after #151 merges with the rebase and the bump). This is the read as the payload prescribes it, run by hand from my seat, so the template's fillability is on record before AC-6's non-author print.

```
v1: 0 of 5 rows passed · ready 2 · unknown 2 · failed 1 · missing: rows 2 and 3 unknown, row 4 failed
views needing love: none yet
```

**Focus (v1) section:** absent from the handoff (as of 2026-10-09 22:21Z, fresh: 42 min old, stale after 36 h)

**v1 rows** — as of 23:05Z · source: the Institution ROADMAP row table's State column → the epics' `Row state:` lines

| Row | Epic | `Row state:` line (verbatim, truncated at the plan) |
|---|---|---|
| 1 | neomjs/neo-agent-institution#351 | row 1 · card half: Mnemosyne (design reads: Clio) · enrollment half: Ada + Emmy · **ready** · 2026-10-09, installed Candidate F |
| 2 | neomjs/neo-agent-institution#477 | row 2 · Euclid (design/provocation: Clio; independent walker: Sophie) · **unknown** · 2026-10-06 · last installed baseline #479: October 3 bundle |
| 3 | neomjs/neo-agent-institution#312 | row 3 · Vega · **unknown** · 2026-10-09, installed Candidate F staged 2026-10-08 23:22Z |
| 4 | neomjs/neo-agent-institution#414 | row 4 · Grace · **failed** · 2026-10-07 (candidate revision not in the report; receipt on #551) |
| 5 | neomjs/neo-agent-institution#424 | row 5 · Ada (walker Vega, accepted 10-05; Euclid alternate) · **ready** · 2026-10-07 16:25Z, candidate C built and installed |

**Views** — as of 23:05Z · source: neomjs/neo-agent-institution#505 receipt comments — none yet (no comment carries a `needs love:` line; the first `design-sweep` receipt writes it)

**Reach and outbound** — unknown (no seat-side source yet; Brain #957's host-edge reader or D#19500's analytics)

**Waits for the operator's word** — as of 23:05Z · search index, may lag — 3 open PRs with an approving review:
- neomjs/neo-agent-institution#653 (Emmy, the detached-seat quit guidance)
- neomjs/neo-agent-skills#151 (this skill's sibling, the design-sweep)
- neomjs/neo-agent-skills#149 (Euclid, Dependabot validation-only)

Remainder: D#19493 (the cut line and the module matrix), the ROADMAP.

**What the print taught the payload:** the `Row state:` lines carry a `plan:` tail that is not state — the screen truncates at it and links the line; the answer line's `missing:` reads the state words, not the plan. Both are in the payload as pushed.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 1e341474-4c8a-4c56-842d-8c6e7c83c100

- 2026-10-09T23:10:00Z @neo-fable-clio referenced in commit `30ad483` - "feat(overview): where v1 stands and which views need love, in one read (#152)

The overview skill: the handoff's Focus (v1) section first, else the rows'
own Row state lines through the ROADMAP's State column, the views' receipt
lines from #505, and what waits for the operator's word; two answer lines;
absent printed as absent. The version bump follows #151's merge.

[skill-growth-justified: new skill graduated from D#19493 OQ-8 (viii); payload-only (router file 870 B, catalog metadata loads at boot per ADR 0008 §3.1); decay clause in the payload head]"
- 2026-10-09T23:10:01Z @neo-fable-clio referenced in commit `d83c144` - "chore(release): bump to 0.1.32 for the overview skill (#152)"
- 2026-10-09T23:10:48Z @neo-fable-clio cross-referenced by PR #153
- 2026-10-09T23:22:42Z @neo-fable-clio referenced in commit `b75ae20` - "fix(overview): count findings not coverage, keep evidence scope, label readiness (#152)

"needing love" counts affirmative governing lines only; "none observed"
is coverage and keys without a line are unassessed. The governing line
per key is the newest of the highest evidence class: a source line never
replaces or resolves an installed one, and prints beneath it labelled.
An approved PR is a candidate until its still-requested reviewers,
checks and mergeability are read; non-PR decisions are named as outside
the read. Six synthetic cases in §3a."
- 2026-10-10T00:01:56Z @neo-fable-clio referenced in commit `f4de927` - "fix(overview): the affirmative readiness label comes only from the merge-readiness projection (#152)

A candidate prints "ready for the operator" only when get_conversation's
merge-readiness projection (the Brain's validateMergeReady) says ready;
a negative verdict prints its reasons, an unavailable one prints
"readiness unverified". The skill computes no readiness of its own; a
field read may add detail beneath a line, never the label. Two more
cases in §3a: an active reviewer hold, and an unavailable projection."
- 2026-10-10T00:18:56Z @tobiu referenced in commit `6c27692` - "feat(overview): where v1 stands and which views need love, in one read (#152) (#153)

* feat(overview): where v1 stands and which views need love, in one read (#152)

The overview skill: the handoff's Focus (v1) section first, else the rows'
own Row state lines through the ROADMAP's State column, the views' receipt
lines from #505, and what waits for the operator's word; two answer lines;
absent printed as absent. The version bump follows #151's merge.

[skill-growth-justified: new skill graduated from D#19493 OQ-8 (viii); payload-only (router file 870 B, catalog metadata loads at boot per ADR 0008 §3.1); decay clause in the payload head]

* chore(release): bump to 0.1.32 for the overview skill (#152)

* fix(overview): count findings not coverage, keep evidence scope, label readiness (#152)

"needing love" counts affirmative governing lines only; "none observed"
is coverage and keys without a line are unassessed. The governing line
per key is the newest of the highest evidence class: a source line never
replaces or resolves an installed one, and prints beneath it labelled.
An approved PR is a candidate until its still-requested reviewers,
checks and mergeability are read; non-PR decisions are named as outside
the read. Six synthetic cases in §3a.

* fix(overview): the affirmative readiness label comes only from the merge-readiness projection (#152)

A candidate prints "ready for the operator" only when get_conversation's
merge-readiness projection (the Brain's validateMergeReady) says ready;
a negative verdict prints its reasons, an unavailable one prints
"readiness unverified". The skill computes no readiness of its own; a
field read may add detail beneath a line, never the label. Two more
cases in §3a: an active reviewer hold, and an unavailable projection."
- 2026-10-10T00:18:57Z @tobiu closed this issue
- 2026-10-10T00:29:30Z @neo-fable-clio cross-referenced by #154

