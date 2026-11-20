---
id: 431
title: framing-governance's cluster table records the README's engine-first top
state: OPEN
labels:
  - bug
  - documentation
  - ai
  - needs-design-authority
assignees:
  - neo-fable-clio
createdAt: '2026-11-20T15:17:47Z'
updatedAt: '2026-11-20T15:18:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/431'
author: neo-fable-clio
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 21564 The README''s first screen shows the engine and names who builds it'
blocking: []
---
# framing-governance's cluster table records the README's engine-first top

No curated labels: the ticket is owned and waits for one merge.

## Context

neomjs/neo#21571 (Resolves neomjs/neo#21564) rebuilds the first screen of `neomjs/neo`'s README. The engine leads; the lead cluster (the category label, the ladder sentence, the apex) opens the README's second part, word for word. The pull request amends ADR 0018 §2.3 in the same diff, and the operator's merge confirms the placement. It was approved on 2026-11-20 (review 5994810300), and that review names this mirror as the follow-up to file before the merge.

## The Problem

`.agents/skills/neo-identity-update/references/framing-governance.md` § Audience-segmented clusters, row 1, says the lead cluster lives on the "README top". After the merge that is false for `neomjs/neo`: its README's first screen projects the Engine, which is §2.3's per-repository projection, and the label opens part two.

The next identity run reads this table as the placement truth. It would classify the engine-first top as drift from the lead cluster and move the label back.

## The Architectural Reality

- `Design authority: "The lead cluster (label, sentence, apex, in that order) opens the README's second part with the label as its heading. The portal and the org profile keep the label on top."` (ADR 0018 §2.3, `learn/agentos/decisions/0018-neo-identity-source-of-truth-model.md` in `neomjs/neo`, as amended by neomjs/neo#21571; in force once that pull request merges.)
- The table sits in a reference that `neo-identity-update` loads on demand, never on every turn.
- ADR 0018 §2.3 is the authority. The table restates its placement for the run, and a mismatch is skill drift. #177 is the precedent for this table.
- Row 2, the Engine cluster, lists `neomjs/neo`'s `package.json`, GitHub description and topics. The README's first screen joins that projection.

## The Fix

In § Audience-segmented clusters:

1. Row 1's "Lives on" names the second part of `neomjs/neo`'s README. The portal and the org profile stay as they are.
2. Row 2's "Lives on" gains the first screen of `neomjs/neo`'s README.

The wording follows §2.3 as merged. No other section changes.

## Acceptance Criteria

- [ ] AC-1: rows 1 and 2 match ADR 0018 §2.3 as merged with neomjs/neo#21571. Every surface the ADR names appears in exactly one row.
- [ ] AC-2: each changed "Lives on" cell names a surface whose live text matches its cluster on `neo` `dev`, checked at authoring time.
- [ ] AC-3: no other section of the reference changes. The pull request's slot rationale states the byte delta and the decay trigger: the table changes only with a placement change, in the same turn as ADR 0018.

## Out of Scope

- The canonical apex section and its concise form.
- The repository description of `neomjs/neo`, which neomjs/neo#21571 lists under "Out of tree" as the operator's.
- An org profile README.

## Avoided Traps

- **Changing the table before the merge.** The merge is the operator's confirmation of the placement. The table follows the record, never the proposal.

## Related

neomjs/neo#21564 · neomjs/neo#21571 · #177 · #175

Blocked until neomjs/neo#21571 merges (native edge: blocked by neomjs/neo#21564).

Live latest-open + closed sweep: the latest 20 open here (38 open in all, orange band; no open ticket or epic owns this table) and an open and closed search on "cluster table README" at 2026-11-20T15:16Z; no equivalent open, and the one closed hit is #177, the precedent, whose Out of Scope does not exclude this. A2A in-flight claim sweep: the latest 30 rows of all read states, 14:22Z to 15:15Z; no claim on this scope. MC sweep: one query on the table naming the README top after a README change; no prior decision beyond #177. Own-assignment sweep: one open ticket of mine here (#10), on another file. Meta-skill sweep: the change stays inside an on-demand reference and touches no router. Structure map: N/A, the script is hosted in the Brain repository; the file and its section exist, and #177 changed the same table.

Origin Session ID: c6de1c6a-2f31-457c-b1e1-0c86ba85b1b3

Retrieval Hint: "framing-governance audience-segmented clusters README top engine-first second part"

📜 Clio · @neo-fable-clio · Claude Fable 5.2 · Claude Code

## Timeline

- 2026-11-20T15:17:47Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-11-20T15:17:49Z @neo-fable-clio added the `bug` label
- 2026-11-20T15:17:49Z @neo-fable-clio added the `documentation` label
- 2026-11-20T15:17:49Z @neo-fable-clio added the `ai` label
- 2026-11-20T15:18:01Z @github-actions added the `needs-design-authority` label
- 2026-11-20T15:18:06Z @neo-fable-clio marked this issue as being blocked by #21564
- 2026-11-20T15:18:46Z @neo-fable-clio cross-referenced by PR #21571
- 2026-11-20T16:44:43Z @neo-fable-clio cross-referenced by #21595
- 2026-11-20T17:39:30Z @neo-opus-vega cross-referenced by PR #21604

