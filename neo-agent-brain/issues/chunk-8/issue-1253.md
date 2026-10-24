---
id: 1253
title: 'The boot focus read: one short business focus block, first in the handoff, derived from the business dashboard'
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-fable-mnemosyne
createdAt: '2026-10-24T20:14:40Z'
updatedAt: '2026-10-24T20:14:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1253'
author: neo-fable-mnemosyne
commentsCount: 0
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
---
# The boot focus read: one short business focus block, first in the handoff, derived from the business dashboard

## Context

The operator, 2026-10-24: when peers start fresh sessions they should read a short CEO dashboard that aligns their technical goals with the business goals; short is crucial. The same evening showed why: the business repository's dashboard, the surface every seat was told to read first, is 28.5 KB (≈ 7k tokens) and grew by 4 KB in one hour; the whole team re-read it after one herd prompt, ten recoveries at once. Measured on one Claude seat, a full recovery reached 125k tokens before any work, 89k of them fixed preload. The graph has the node types for this since July (#14446 delivered `BUSINESS_GOAL`/`METRIC` under #123) and no populated goal node: two `search_nodes` sweeps tonight found the schema, the `business-goal` concept nodes and nothing else. #957 (Clio) owns the `## Focus (v1)` handoff section for v1 rows, views and reach; nothing owns the business half or the size.

## The Problem

A fresh session has no cheap, authoritative answer to "what does the business need from me this week". The long answer exists (the private dashboard, the decision record, the week plan: ~65 KB together) and is right for the lead; it is wrong as a boot read for ten seats, and it is private to seats with the business clone. The Sandman handoff is read by every seat and rendered in the Fleet Manager's Golden Path pane, yet it carries graph gaps and the computed Golden Path only (issues; the launch playbook first) and nothing about the money events, the week's outcomes or the Nov 30 rule. Without the block, the Golden Path keeps recommending better code to a company whose stated problem is invisibility.

## The Architectural Reality

- Producer: `ai/services/graph/GoldenPathSynthesizer.mjs` writes sections; the handoff's reader is `get_sandman_handoff` (`sandmanHandoffStore.mjs:27`); the Fleet source extracts sections by heading; the `/overview` skill already expects a Focus section (#957's AC-6).
- #957's design: `## Focus (v1)` with four blocks, inputs through a host-edge reader with an `unknown` shape per block, projection under #123's `publicFlag` rule. This ticket adds the **business block** and the **cap**; it does not rewrite #957.
- Schema: `businessSchema` (`BUSINESS_GOAL`/`METRIC` with `{claimClass, falsifyingQuery, windowSemantics, confoundDisclaimer, publicFlag}`), delivered by #14446; `publicFlag` travels with the node.
- Boundary (#123, D#14430): mechanism public; goals, targets, revenue and client specifics private. The handoff lives on the plane's private data root; the public corpus projection must never carry the block.
- The readers already planned: the recovery runbook's step that reads the operator's focus artifact (Clio's outcome-8 leaf of the Oct 26 week plan) and the AGENTS.md boot rule that today points seats at the dashboard file.
- Design authority: none challenged; the handoff format is additive (ADR 0033's boundary).

## The Fix

1. **One source, one writer.** The lead keeps `strategy/FOCUS.md` in the business repository: at most 1,200 bytes (≈ 300 tokens), a hard cap. Content: the question of the day and its decision rule with the date; the money events' next dates (≤ 3 lines); the week's outcomes, one line each with the owner; the dates of the week; one link to the long dashboard. No client names, no prices (categories and dates only, per #123).
2. **The node.** A host-edge writer (the business clone is host-local, like #957's GitHub inputs) upserts the block as the first populated `BUSINESS_GOAL` node, id `business-focus`, `publicFlag: false`, `windowSemantics` = the week, `falsifyingQuery` = the dashboard commit it was derived from. The graph now has the business in it; Golden Path v2 (#122) can rank against it later; nothing ranks on it here (ADR 0033: reporting, additive).
3. **The section.** The synthesizer renders the node as `## Focus (business)` **as the first section of the handoff**, before the gap audit, with the source commit and the as-of time; when the file is absent or over the cap the section says so (`unknown: no FOCUS.md`, `over-cap: 1,312 bytes, first 1,200 rendered`) rather than rendering nothing. With the sibling `get_sandman_handoff` filing, the section is one addressable slug (`focus`) and a bare read returns the route plus this block.
4. **The readers.** The recovery runbook reads the section first (Clio's leaf; this ticket hands it the surface); the AGENTS.md boot rule points at the section instead of the 28 KB file (a one-line neo-agent-skills follow-up, filed when this lands); `/overview` prints it; the Fleet pane renders it with #957's envelope field.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `strategy/FOCUS.md` (business repo, private) | the lead (one writer, the dashboard's rule) | ≤ 1,200 bytes; the five content lines above; derived from the dashboard's top block | over the cap → the writer sees `over-cap` in the handoff; the first 1,200 bytes render | the file's own header | the cap asserted by the host-edge writer's unit arm |
| `BUSINESS_GOAL` node `business-focus` | `businessSchema` (#14446), #123's boundary | upserted by the host-edge writer with `publicFlag: false` | no file → no node write, the previous node stays with its as-of | schema docs | unit arm: the node validates; a negative projection test: the public corpus projection excludes it |
| `## Focus (business)` handoff section | the synthesizer (producer-owned) | first section; as-of + source commit; `unknown`/`over-cap` shapes | synthesizer runs without the node → the section says `unknown` | synthesizer section JSDoc | snapshot arm with and without the node |
| `get_sandman_handoff` slug `focus` | the sibling filing | returned with the default read | — | — | the sibling's arms |

Decision Record impact: aligned-with ADR 0033 (additive reporting); depends-on #14446's schema; none amended.

## Acceptance Criteria

- [ ] AC-1 A `FOCUS.md` of ≤ 1,200 bytes becomes a `BUSINESS_GOAL` node `business-focus` with `publicFlag: false`; a 1,300-byte file produces the node with `over-cap` recorded and the first 1,200 bytes (unit arms).
- [ ] AC-2 The handoff's first level-two section is `## Focus (business)` carrying the block, the as-of time and the source commit; absent node → the section reads `unknown` with the reason (snapshot arm).
- [ ] AC-3 The negative projection test: the public corpus projection never contains the section or the node.
- [ ] AC-4 The section costs under 400 tokens in a bare `get_sandman_handoff` read (byte count asserted).
- [ ] AC-5 (post-merge) one fresh session of another family reads the section first and names its week row from it without opening the business dashboard (receipt on this ticket); the runbook and boot-rule follow-ups are filed with their numbers here.

## Out of Scope

#957's v1/views/reach blocks and its host-edge GitHub reader; any ranking change (#122); the Fleet pane leaf (Institution #312); metrics ingestion (#123 Leaf 3); the public projection of any business number.

## Avoided Traps

- Putting the block on the context frontier: measured at 160 KB tonight, it cannot be a boot surface.
- A second writer: the block is derived from the dashboard's top block by the lead; peers send state lines, never edits.
- A cap enforced by prose: the writer measures bytes and records `over-cap`; a seat never reads a truncated block without knowing it.
- The dashboard itself as the node body: 28 KB of record is the long form, linked once.

## Related

Parent #123 (the business engine; this is the first populated goal node); sibling #957 (the Focus (v1) section — same handoff, different blocks; the cap and the ordering are proposed to its author by comment tonight); #14446 (the schema, done); #122 (ranking later); the sibling filings on the boot cost tonight (handoff default, frontier default, mailbox projection, skills description budget); the Oct 26 week plan's outcome 8 (Clio's runbook leaf); the business repository's dashboard READ THIS FIRST block (private).

Live latest-open sweep: the latest 20 open issues, created-descending, at 2026-10-24 19:5xZ; no equivalent (#957 is the closest and is linked, not duplicated). A2A in-flight claim sweep: the latest 30 `[lane-claim]` rows, none on this scope. MC sweep: three problem-noun queries (handoff payload, frontier payload, boot cost), 6 results each, initialization noise, no prior decision found; the June concept and D#14430 are the record and are cited. Own-assignment sweep: 11 open, none overlapping (#122 and #117 are mine and are the ranking layer this ticket stays out of). Structure map: `ai/services/graph` (the synthesizer and its section modules), `ai/services/memory-core` (the graph writers); the host-edge writer follows #957's reader pattern — structural pre-flight runs at authoring.

Origin Session ID: 542d408d-b0eb-497c-8f57-a19ca5833f5a

Retrieval Hint: "business focus block handoff first section BUSINESS_GOAL node publicFlag FOCUS.md 1200 bytes boot read"

## Timeline

- 2026-10-24T20:14:40Z @neo-fable-mnemosyne assigned to @neo-fable-mnemosyne
- 2026-10-24T20:14:41Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-10-24T20:14:41Z @neo-fable-mnemosyne added the `ai` label
- 2026-10-24T20:14:42Z @neo-fable-mnemosyne added the `architecture` label
- 2026-10-24T20:14:42Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-10-24T20:16:14Z @neo-fable-mnemosyne added parent issue #123
- 2026-10-24T20:17:12Z @neo-fable-mnemosyne cross-referenced by #957

