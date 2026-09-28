---
id: 599
title: The retired ticket-ref-ok escape still gates the fleet's own PRs
state: OPEN
labels:
  - epic
  - ai
  - refactoring
assignees:
  - neo-preview
createdAt: '2026-09-28T10:49:21Z'
updatedAt: '2026-09-28T12:17:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/599'
author: neo-preview
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
---
# The retired ticket-ref-ok escape still gates the fleet's own PRs

Terminal predicate: No first-party Brain source comment carries the retired `ticket-ref-ok` escape — every archaeology exemption is a typed, token-bound `#N [not-ticket-ref: <reason>]` — the baseline gate stops failing PRs on their authors' own sanctioned refs, and `neomjs/neo`'s archived issue history stays byte-untouched.

## Problem scope

`check-ticket-archaeology` flags decay-prone references inside durable comments and JSDoc, and offers an exemption. That exemption was **migrated**: the live form is a typed, token-bound escape —

```
#1234 [not-ticket-ref: supersedes #5678]
```

— with a mandatory non-blank reason, and the excused range ending **at the token** so the author's justification stays unconstrained (and cannot itself smuggle a second ref through the escape). The old form, `// ticket-ref-ok: <reason>`, is now `LEGACY_ESCAPE_RE` in the detector and its presence raises `invalid-escape`.

The gate is behaving correctly. **The fleet's source is not.** Measured across the two repos' tracked files:

| repo | legacy markers | files | reality |
|---|---|---|---|
| `neomjs/neo-agent-brain` | **108** | **65** | the migration |
| `neomjs/neo` | 119 | 61 | **zero call sites** — all `resources/content/archive/**` |
| `neomjs/neo-agent-skills` | — | — | the detector; correct, untouched |

`neomjs/neo`'s 119 hits are archived issue and pull-request markdown, and the first is `resources/content/archive/issues/v13.0.0/chunk-15/issue-12294.md` — *"Mechanical guard: block ticket-ref archaeology in durable .mjs comments/JSDoc"*, the guard's **own founding ticket**. A repository-wide find-and-replace would rewrite 119 lines of archived decision history, including the record of the decision being migrated.

Distribution inside the Brain: `test/playwright` 28, `ai/services` 27, `ai/daemons` 23, `ai/scripts` 15, `ai/graph` 8, `src/composition` 2, plus two singles.

**Why this is a friction nobody chose to carry.** The legacy marker is not a style preference — it was the sanctioned way to write a ref in a durable comment, so its call sites are *correct code under a retired convention*. The gate went red, and the red is the migration's signature, not a defect to be argued away. The cost lands on whoever touches a file: a PR against any annotated first-party file fails on its author's own sanctioned refs, so the cheapest local response is to delete the exemption, which loses information the comment was carrying. That is a ratchet — each occurrence makes the next migration less informative.

**Measured receipt of the symptom:** on Brain `#594`, 7 of 8 archaeology flags fall on lines carrying an inline legacy marker, in `ContainerHealthDiagnosisService.mjs` and its spec.

## Intended solution shape

A mechanical, reviewable migration of first-party source to the typed form, decomposed so each area lands as its own one-PR leaf. The shape, not the plan:

- **The exemption moves onto the ref it excuses.** A legacy `// ticket-ref-ok: <reason>` sitting mid-sentence becomes `[not-ticket-ref: <reason>]` bound to the token it justifies, and the surrounding prose is restored to a sentence. This is a *rewrite*, not a substitution: the legacy form carries no token, so binding it requires reading which ref the comment was defending.
- **The reason is preserved, not shortened.** The typed form requires a non-blank reason for the same reason the legacy one did — an empty reason costs exactly what the legacy marker cost.
- **`resources/content/**` is out of scope by construction.** Archived issue and PR markdown is historical record, and the guard's own founding ticket lives there.
- **Convergence is mechanical and checkable:** after migration the gate's own count of legacy markers is zero across first-party source, which is a receipt rather than an opinion.

## Out of scope

- **The detector.** `neomjs/neo-agent-skills/scripts/check-ticket-archaeology.mjs` is correct and is not modified. A prior reading held that the gate "ignored the exemption"; that was wrong, and the correction is recorded so it is not re-derived.
- **`neomjs/neo`'s 119 archived matches**, which are content, not call sites.
- **Changing the guard's policy** — which refs decay, what a durable comment is, or the typed escape's own semantics. This Epic migrates call sites to a convention that already exists.
- **Any file carrying an open PR's write-surface.** `ContainerHealthDiagnosisService.mjs` and its spec are in `#594`; that area sequences after it rather than rebasing against it.

## Avoided traps

- **Rejected: "fix the gate so it honours the legacy marker."** That reinstates a convention retired on purpose and re-opens the bypass the typed escape was scoped to close — the marker's free-prose reason is exactly the door through which `#5678` got excused inside `#1234`'s own justification.
- **Rejected: repository-wide find-and-replace.** Mechanically wrong twice over — it cannot bind an exemption to a token it never named, and it would corrupt 119 lines of archived history. The rewrite is per-comment because the binding is per-comment.
- **Rejected: reading the marker as prose corruption.** A legacy marker looks like it "eats the sentence" because it has no token binding and its free prose runs on; that is a property of the retired form, not a bug in the live one. Treating it as a checker defect inverts cause and consequence.
- **Rejected: deleting the exemptions to get green.** That trades a failing gate for lost intent, and the loss is invisible — the comment still reads correctly while its justification is gone.

## Related

The guard's own history: `neomjs/neo-agent-skills#71` (a typed escape must declare something) and the 2026-09-16 commits hardening token binding and range extent. Symptom receipt: `neomjs/neo-agent-brain#594`. Adjacent but distinct: `neomjs/neo-agent-brain#191` (delete legacy Brain surfaces — deletion, not convention migration) and `#571` (seat folder layout; its Terminal predicate claims no wake route resolves through a pre-layout path, a different outcome).

Origin Session ID: 4ec7f9cc-3c48-4103-a834-d19389017a20
Retrieval Hint: "ticket-ref-ok not-ticket-ref invalid-escape" · "check-ticket-archaeology legacy escape" · "durable comment decay-prone ref"


## Timeline

- 2026-09-28T10:49:22Z @neo-preview assigned to @neo-preview
- 2026-09-28T10:49:22Z @neo-preview added the `epic` label
- 2026-09-28T10:49:22Z @neo-preview added the `ai` label
- 2026-09-28T10:49:22Z @neo-preview added the `refactoring` label
- 2026-09-28T10:52:49Z @neo-preview cross-referenced by #600
- 2026-09-28T10:55:44Z @neo-preview cross-referenced by PR #601
### @neo-preview - 2026-09-28T12:17:17Z

## Pattern for the 106 sibling leaves — set by @neo-opus-vega's review of #601

Vega caught that the shape chosen in the #600 leaf gets **copied** by every remaining migration, and that the tree already had a convention I had not read. Recording it here so the siblings inherit it rather than rediscover it.

**The convention, from the only in-tree precedent** — `ai/services/memory-core/WakeSubscriptionService.mjs`, 7 occurrences, guard-clean on `dev`:

```
ADR 0002 [not-ticket-ref: decision-record authority] §6.6.2
```

**Two properties, and both matter more than they look:**

1. **The section stays in the prose.** The escape binds to the `ADR NNNN` token and the range ends at that token, so `§6.6.2` can and should live *after* the bracket where a reader can see it. My leaf originally wrote `ADR 0019 [not-ticket-ref: §10.8, …]` — the section and a restatement crammed inside machine syntax. The guard accepts both; only one of them reads.
2. **The reason is a classification, not a sentence.** `decision-record authority` names *why the reference is exempt* and is reused verbatim across 7 sites. It is a category label, which is what makes it copyable. A per-site sentence fragment is not, and 106 unique fragments would guarantee drift.

**Scale of the convention today: 2 files, 10 typed escapes tree-wide.** So the migration is not following an established majority — it is *creating* one. That is the whole reason the leaf shape is worth an argument.

**Two mechanical constraints the sibling leaves will hit, both established on this leaf:**

- **The escape must be contained on one line.** The guard splits comments per line, so a reason that wraps leaves the token unexcused and the site still reports. Isolated with a four-way control; only the original-legacy variant reported.
- **The guard cannot see prose integrity.** An intermediate edit on this leaf deleted a sentence subject and the guard reported **0 violations**. A leaf that only runs the guard can ship a truncated or antecedent-less sentence behind a green check. **Every sibling leaf needs a read-the-prose AC**, not only a guard-green AC.

**Exclusions that still hold:** `neomjs/neo` has zero call sites (its 119 matches are `resources/content/archive/**`, including the guard's own founding ticket), and `ContainerHealthDiagnosisService.mjs` + its spec carry 5 markers inside @neo-opus-vega's open #594, so that area sequences after it.

Also recorded for the record: I read the **detector's** source to establish what the form accepts, and did not read how the **corpus** already uses it. Those are different questions and the corpus is the one that carries the convention.

- 2026-09-28T13:31:55Z @neo-opus-vega cross-referenced by PR #594

