---
id: 334
title: 'unit-test reads 168 KB before any spec work, whatever the work is'
state: CLOSED
labels:
  - enhancement
  - ai
  - testing
  - performance
  - model-experience
assignees:
  - neo-opus-vega
createdAt: '2026-11-10T20:36:14Z'
updatedAt: '2026-11-10T22:04:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/334'
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
closedAt: '2026-11-10T22:04:54Z'
---
# unit-test reads 168 KB before any spec work, whatever the work is

## Context

Session `73b54f9f-8720-4b8f-b266-add51b1f613f` (2026-11-10) added two arms to an existing spec. That activated `unit-test`, and §2 of `.agents/skills/unit-test/references/unit-test.md` asked for seven files first. Counted from that session's transcript, every tool result once, those seven reads were 185k of the 1,236k characters that entered the context: 15 %. The files of the change itself were 5 %.

The seven files at `neomjs/neo@0d1170fe22`:

| File | Lines | Bytes |
|---|---|---|
| `src/Neo.mjs` | 1,318 | 50,518 |
| `src/core/Base.mjs` | 1,397 | 53,648 |
| `test/playwright/setup.mjs` | 223 | 8,638 |
| `learn/guides/testing/UnitTesting.md` | 413 | 17,559 |
| `test/playwright/unit/vdom/RealWorldUpdates.spec.mjs` | 276 | 10,789 |
| `test/playwright/unit/collection/Base.spec.mjs` | 453 | 21,212 |
| `test/playwright/unit/core/Effect.spec.mjs` | 200 | 5,856 |
| **Total** | **4,280** | **168,220** |

By arithmetic at 4 bytes per token that is about 42k tokens: 4 % of a 1M window and 16 % of a 258k one.

## The Problem

§2 opens with "After this workflow is activated" and scales with nothing. `SKILL.md` activates the workflow for writing, modifying, fixing and running unit tests, so a seat that runs one spec for a receipt owes the same 168 KB as a seat that designs a new harness pattern. A read made early is also carried through every later turn of the session.

Step 1 of §2 boots a session inside a running one: a Memory Core healthcheck, `get_all_summaries({limit: 5})`, and an `add_memory` for "the initialization turn". The last one contradicts `§memory_core_protocol`, which asks for one consolidated save at the end of the turn.

## The Architectural Reality

- `SKILL.md` is the router and `references/unit-test.md` the payload. §1, the Review-Only Boundary (`neomjs/neo#16068`), already split the trigger once: a review that inspects test diffs does not activate the author path.
- §2's reads are older than this repository (`neomjs/neo@ffa0c56e70`, 2026-04-29) and arrived unchanged with the seed (`243157f`). Step 1 got its "Maintainer only" qualifier in `#310`.
- §4 states the failure modes the reads protect against: the two imports, unique `className`s, manual VDOM init, connect-on-init singletons. It stays.
- `neomjs/neo` holds 628 spec files under `test/playwright/unit/`, so nearly every subject has a sibling spec.
- Design authority: "Measure bloat against the smallest-context peer (~258k tokens), not your own — the substrate tax is paid by the leanest harness, on every load." (`.agents/skills/create-skill/references/skill-authoring-guide.md`). The same guide quotes the rule for workflow files: "the bare always-relevant minimum is in there. and edge cases as ONE LINE triggers."
- Prescription checked: `.agents/skills/unit-test/references/unit-test.md` §2 — owns the concern. The router's trigger is right as it is; the reads are mandated in the payload.

## The Fix

Rewrite §2 so the reads follow the work (ADR 0007 dispositions: `rewrite` for the reads, `retire` for step 1):

| The work | Read first |
|---|---|
| Running specs | nothing beyond the workflow file |
| Arms in, or a fix to, an existing spec | that spec, whole: it is the example |
| A new spec file | the closest sibling spec (the subject's directory, else its base class) and `test/playwright/setup.mjs` |
| A pattern no sibling shows, or a change to the harness | also today's full set |

Step 1 goes, and the audience note at the top loses "private-service initialization".

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `unit-test.md` §2, the reads (skill-loaded, both audiences) | `skill-authoring-guide.md`, the 258k rule; ADR 0007 §5.4 | reads keyed on the work; the full set only for a new pattern | a seat unsure of its row takes the next heavier one | the section itself | PR body: bytes per row at the Engine's `dev` head |
| `unit-test.md` §2 step 1 | `§memory_core_protocol`: one save per turn | removed | none needed: the turn-start mailbox check and the turn-end save already run | the audience note | the diff |

## Decision Record impact

aligned-with `neomjs/neo#adr:0007-agents-md-compaction-taxonomy` (§5.4: rewrite or retire before adding).

## Acceptance Criteria

- [ ] §2 names the reads for the four kinds of work above, and the full set stays reachable for a new pattern.
- [ ] No Memory Core call remains in §2, and the audience note matches.
- [ ] `unit-test.md` is no larger than at `a28885c`.
- [ ] `npm run lint` passes, and each runner of `npm test` passes when run on its own (`#37`).
- [ ] The PR body lists the bytes per row, measured at the Engine's `dev` head.

Falsifier, not a criterion: a session that needed a file its row does not name reports it as a `defect-note:`. A second such note reopens the table.

## Out of Scope

- §3 to §7 of the workflow, and the Unit Testing guide itself.
- `whitebox-e2e`: its §1 is a Neural Link exploration, a different cost.
- The size of the other workflow payloads.

## Avoided Traps

- **"Read once per session" alone.** Sessions rotate, and the first activation still pays 168 KB.
- **Line ranges into `src/Neo.mjs`.** They rot with the next commit.
- **Deleting the full set.** The first spec of a new kind needs it.

## Related

- #326 (the loaded rules read the mailbox and memories at full size)
- #329 (the structure-map gate prints 98.5 KB per run)
- #37

Live latest-open + closed sweep: checked latest 20 open, created-descending, and closed searches (unit-test; initialization reads) at 2026-11-10T20:36Z; no equivalent or prior exclusion found. 31 open (orange): no owning ticket or epic for this.
A2A in-flight sweep: latest 30 messages, all read states, at 20:36Z; no claim on this scope.
Memory Core rationale sweep: "unit-test workflow initialization mandatory reads every activation cost"; no prior decision found.
Own-assignment sweep: `#5`, `#6`, `#302`; none on this surface.
Structure map: N/A. `npm run ai:structure-map -- --files --loc` exits 0 (98,730 B); this is an edit in place, no file is added or moved.

Origin Session ID: b1fcc0fd-3006-429b-bb67-69b4591de962
Retrieval Hint: "unit-test workflow initialization reads seven files on every activation"

## Timeline

- 2026-11-10T20:36:14Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-10T20:36:16Z @neo-opus-vega added the `enhancement` label
- 2026-11-10T20:36:16Z @neo-opus-vega added the `ai` label
- 2026-11-10T20:36:16Z @neo-opus-vega added the `testing` label
- 2026-11-10T20:36:16Z @neo-opus-vega added the `performance` label
- 2026-11-10T20:36:17Z @neo-opus-vega added the `model-experience` label
- 2026-11-10T20:48:54Z @neo-opus-vega cross-referenced by PR #337
- 2026-11-10T22:04:54Z @tobiu referenced in commit `4639a8f` - "fix(unit-test): reads follow the work (#334) (#337)

Section 2 asked for seven files, 168 KB, on every activation, also for
running one spec. It now names the reads per kind of work and keeps the
full set for a pattern no sibling spec shows. Step 1, a session boot
inside a running session, is removed."
- 2026-11-10T22:04:55Z @tobiu closed this issue

