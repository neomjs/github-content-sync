---
id: 177
title: framing-governance's cluster table records the lead cluster after neomjs/neo#19826
state: CLOSED
labels:
  - bug
  - documentation
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-16T19:40:54Z'
updatedAt: '2026-10-16T20:19:44Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/177'
author: neo-opus-ada
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
closedAt: '2026-10-16T20:18:16Z'
---
# framing-governance's cluster table records the lead cluster after neomjs/neo#19826

## Context

neomjs/neo#19826 merged at `8c071e0934` (2026-10-16 19:03:58Z). It implements Discussion neomjs/neo#19789 with the operator's placement of the lead cluster. The README top, the portal (title, meta and Open Graph descriptions, ld+json, hero) and the `llms.txt` header now open with the category label. Behind it stands the apex as the proof paragraph. ADR 0018 §2.3 records the rule:

> a category label leads README, the portal and the org profile: *Neo.mjs is an agent institution: persistent AI maintainers with identities, memory, peer review, and a record anyone can read.* Behind it, the apex (OD-1) is the proof paragraph.

The same §2.3 adds the per-repository projection: each repository's own metadata projects its own product, so `neomjs/neo`'s projects the Engine.

## The Problem

`.agents/skills/neo-identity-update/references/framing-governance.md` § Audience-segmented clusters still records the old placement:
- **Row 1, the Engine cluster,** lives on `package.json`, the GitHub description, `index.html`, the hero and the `llms.txt` header. The last three now present the whole and lead with the label.
- **Row 2** lives on the README top and ROADMAP. The README top leads with the label now, and ROADMAP carries no framing line: it holds "the next-release scope only".
- **The lead cluster has no row.**

The next identity run reads this table as the placement truth. It would classify the portal's label as drift from the Engine cluster and "fix" it back.

## The Architectural Reality

- The table sits in a reference that `neo-identity-update` loads on demand, never on every turn.
- ADR 0018 §2.3 is the authority. The table restates its placement for the run, and a mismatch is skill drift.
- Row 2's Brain and Institution descriptions are correct as they stand. They project their own products, the Agent OS and the Fleet Manager, in the "AI engineering team" register.

## The Fix

In `framing-governance.md` § Audience-segmented clusters:
1. A first row for the lead cluster: the category label, then the ladder sentence. It lives on the README top, the portal (title, descriptions, ld+json, hero, `llms.txt` header) and the org profile, for first contact by people and answer engines.
2. Row 1 narrows to `neomjs/neo`'s own metadata: `package.json`, the GitHub description and topics.
3. Row 2 keeps the Brain's and Institution's repository descriptions. The README top and ROADMAP leave it.
4. The paragraph under the table counts four clusters and names the label's relation to the apex: the label leads, and the apex stands behind it as proof.

## Acceptance Criteria

- [x] AC-1: the table's rows match ADR 0018 §2.3 as merged in neomjs/neo#19826. Every surface the ADR names appears in exactly one row.
- [x] AC-2: every "Lives on" cell names surfaces whose live text matches that row's cluster on `neo` `dev` or the live repository metadata, checked at authoring time.
- [x] AC-3: no other section of the reference changes. The PR's slot rationale states the byte delta and the decay trigger: the table changes again only with the operator's next placement change, in the same PR as ADR 0018.

## Out of Scope

- The canonical apex section and its concise form. The apex is unchanged, and ADR 0018 §2.3 keeps "OD-1's concise form and keyword cluster" for the surfaces that present the whole.
- The neo repository description and the org description, which wait for the operator (neomjs/neo#19811's read-back).

## Related

neomjs/neo#19811 · neomjs/neo#19826 · neomjs/neo#19789 · #175 · #176

Live latest-open sweep: the latest 20 open issues here at 19:39Z, no equivalent; a search for "cluster table" found nothing. A2A sweep: the latest 25 messages, all read states; no claim on this scope. MC sweep: this session's identity-run records; no prior decision on the table beyond #176. Own-assignment sweep: no open ticket of mine here on this file.

Origin Session ID: ac1f92da-06ba-4c59-8e6e-25a2d1397d69

Retrieval Hint: "framing-governance audience-segmented clusters lead cluster agent institution label"


## Timeline

- 2026-10-16T19:40:55Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-16T19:40:56Z @neo-opus-ada added the `bug` label
- 2026-10-16T19:40:56Z @neo-opus-ada added the `documentation` label
- 2026-10-16T19:40:56Z @neo-opus-ada added the `ai` label
- 2026-10-16T19:46:53Z @neo-opus-ada cross-referenced by PR #178
- 2026-10-16T20:18:17Z @tobiu closed this issue
- 2026-10-16T20:18:18Z @tobiu referenced in commit `0423b9c` - "docs(neo-identity-update): the cluster table records the lead cluster after neomjs/neo#19826 (#177) (#178)

The category label leads the README top, the portal and the org profile (ADR 0018 §2.3), so those surfaces get their own row; the Engine row narrows to neomjs/neo's own metadata, and row 2 keeps the Brain's and Institution's descriptions. Release 0.1.43."

