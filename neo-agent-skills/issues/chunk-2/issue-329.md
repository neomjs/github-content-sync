---
id: 329
title: 'The structure-map gate prints the whole tree: 98.5 KB per run'
state: OPEN
labels:
  - enhancement
  - good first issue
  - ai
  - performance
  - needs-design-authority
assignees: []
createdAt: '2026-11-09T23:59:52Z'
updatedAt: '2026-11-10T00:00:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/329'
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
# The structure-map gate prints the whole tree: 98.5 KB per run

## Context

Split out of #326 on 2026-11-10: this row was added to that ticket after its claim, so it gets its own leaf and #328 closes #326 as claimed. Same finding class: a loaded rule names a command without its size parameter.

## The Problem

Three gates tell a seat to run `npm run ai:structure-map -- --files --loc` and to cite the owning folder or record N/A. The command prints the whole Brain tree: 98,552 B and 4,543 lines at Brain `4605d893` (measured 2026-11-09), roughly 28k tokens, which is about 11 % of a 258k-token window. A seat needs the rows of one folder. A ticket or review that places no file needs none.

## The Architectural Reality

Call sites at `origin/dev` plus #328:

- `.agents/skills/ticket-create/references/ticket-create-workflow.md` §1c (24,977 B after #328, against a 25,000 B per-file budget, so a longer command has to be paid for inside the file).
- `.agents/skills/epic-create/references/epic-create-workflow.md`, Procedure step 2.
- `.agents/skills/pr-review/references/pr-review-guide.md` §2 step 7 (under the combined byte gate of §6.3).
- `.agents/skills/tech-debt-radar/references/tech-debt-radar-guide.md:15` uses the same command for a whole-tree measurement; that one may be right as it is.

The script already bounds its output: `-r, --root <path>` (neo-agent-brain `ai/scripts/diagnostics/structureMap.mjs:57`).

Design authority: n/a. The gates work; this leaf bounds what they load.

## The Fix

In the three gates the command names `--root <owning folder>`; the whole-tree run stays for a ticket or PR that places a new top-level folder. Each gate keeps its "never `--silent`" clause. The radar's call site is audited and either keeps the whole tree with one clause saying why, or takes a root.

The implementer runs `turn-memory-pre-flight` and `create-skill` first.

## Acceptance Criteria

- [ ] The three gates name `--root`; the PR body shows the output size of one bounded run beside the 98,552 B.
- [ ] `ticket-create-workflow.md` stays within its per-file budget and `lint-skill-corpus` passes.
- [ ] The radar's call site carries its decision in the PR body.

## Out of Scope

Changing the script, and removing the gate for tickets that place no file (a design question for the gates' owners).

## Related

#326 and #328 (the sibling finding), neomjs/neo-agent-brain#1555.

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Live latest-open + closed sweep: newest open issues and a search over all states (structure-map, "structure map") at 2026-11-09T23:59Z; no equivalent found; 32 open (orange), no owning ticket for the gates' output size. A2A in-flight claim sweep: newest rows at 23:57Z, no claim on this scope. MC sweep: run for #326 ("recovery reads too many tokens"), 5 results, no prior decision. Own-assignment sweep: #191 here, not overlapping. Structure map: N/A, no file is placed.

Origin Session ID: 6bc6e94b-0b0c-4f41-ad1f-ec6dcdbfdf6d

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session 6bc6e94b-0b0c-4f41-ad1f-ec6dcdbfdf6d

## Timeline

- 2026-11-09T23:59:54Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-09T23:59:54Z @neo-fable-mnemosyne added the `good first issue` label
- 2026-11-09T23:59:54Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-09T23:59:55Z @neo-fable-mnemosyne added the `performance` label
- 2026-11-10T00:00:04Z @github-actions added the `needs-design-authority` label
- 2026-11-10T00:01:05Z @neo-fable-mnemosyne cross-referenced by #326
- 2026-11-10T00:02:26Z @neo-fable-mnemosyne cross-referenced by PR #328

