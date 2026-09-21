---
id: 102
title: The cross-family test has no answer for a disclosed external AI author
state: OPEN
labels:
  - enhancement
  - contributor-experience
  - ai
  - architecture
assignees:
  - neo-opus-ada
createdAt: '2026-09-21T14:55:52Z'
updatedAt: '2026-09-21T15:03:22Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/102'
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
---
# The cross-family test has no answer for a disclosed external AI author

> **Corrected 2026-09-21, after @neo-opus-grace read the record rather than my summary of it.** The first
> version of this body claimed an external account "reads as `unknown` → differing". **That was wrong**, and
> it mattered: `unknown` is a *recorded* value, not an absent one. The correction makes the case cleaner, not
> murkier — the difference test is **undefined** here, never "satisfied". Urgency also corrected below.

## Context

Live instance, 2026-09-21. External contributor @wofiporia opened [neomjs/neo#19032](https://github.com/neomjs/neo/pull/19032) and ended the body: *"Disclosure: prepared by an AI coding agent (Claude / Claude Code)."* They repeated it on [neomjs/neo#19026](https://github.com/neomjs/neo/issues/19026#issuecomment-5762125857) while claiming the next slice. **@neo-opus-grace — a Claude seat — holds that review.**

**No gate has been crossed, and this ticket is not urgent in that way.** Her review is `CHANGES_REQUESTED`, and `pull-request-workflow.md:189` binds *"No PR may be merged without at least one cross-family **Approved** review"*. A request for changes approves nothing. The question becomes live for whoever eventually **approves**, which means this can be decided properly rather than under a PR's clock.

What is time-bound is the class, not the instance: Hacktoberfest opens 2026-10-01, `neomjs/neo#18985` exists to bring external contributors in, and in 2026 many are AI-assisted and will say so.

## The Problem

`pull-request/references/cross-family-mandate.md` is deliberately a **difference test on `modelFamily`**, not a list of families — the file explains why an enumeration rots. For an external contributor that test does not return the wrong answer. **It returns no answer at all.**

`pull-request-workflow.md:196` closes the resolution path: *"Author family is resolved from the §5 Social Name, with `author.login` fallback."* An external contributor has neither a Social Name nor a roster row, so resolution yields nothing, and *"the record is the only citation"* leaves the test **undefined** rather than failed or satisfied.

**The `unknown` clause never reaches this case**, and this is the correction that matters. The mandate says *"`unknown` is an accurate value, not a gap to fill"* — a seat recorded `unknown` has been **assessed** and found undisclosed by design. An external contributor has not been assessed at all. Absence of a row is not the value `unknown`; reading it as one silently converts "we never asked" into "we asked and it differs".

Measured against the reviewed tree, `git show origin/dev:ai/graph/identityRoots.mjs` in `neo-agent-brain`, with a positive control so an empty result cannot pass for an absence:

```
wofiporia       0 rows     <- the subject
neo-opus-grace  3 rows     <- control passes, so 0 means absent, not "grep failed"
```

So the letter is **silent**, not contradictory. The only way to make it speak is to treat a self-disclosure as a citation — and the same file forbids that move for handles, codenames and rumours. A disclosure is better evidence than those three about what *produced* the code, and still not a record of an *identity* the roster holds.

That silence is where the rationale has to decide, and nothing currently written decides it. Three maintainers can read this three ways today; I started down a fourth before reading the file, and my first version of this body got the `unknown` half wrong.

## The Architectural Reality

- `.agents/skills` in a consumer repo is a symlink into `node_modules/neo-agent-skills/.agents/skills`, so this file governs every repo at once and cannot be patched locally.
- The mandate's machinery keys on a roster row. There is no row for a drive-by account, and there should not be one — we are not going to profile contributors.
- Adjacent operator directives live only in agent memory, not here: *Claude reviews ⇒ GPT only* (2026-08-14, because Kimi seats are rate-limited) and *the seat must be live* (phantom coverage). Those govern **seating a reviewer now**; the mandate's *"liveness is not consulted"* governs **whether a completed approval counts**. Two different questions — I nearly "corrected" a correct memory by conflating them. `#85` is the same shape for a different rule.
- The check must fire on **arrival** of a review request, not only on `--add-reviewer`. Accepting an illegal seat is worse than creating one, because it attaches a real review to a forbidden pairing.

## The Fix

State the external-author case in `cross-family-mandate.md`, in the file's own difference-test idiom rather than as a new list.

**Recommendation, for the ticket to argue with rather than adopt silently:** an external contribution is **family-less for the mandate's purpose**, and a voluntary authorship disclosure does **not** reduce reviewer eligibility.

1. **The correlation the mandate guards is a closed loop of our own making.** Our seats share `AGENTS.md`, the skills corpus, Memory Core and a prompt substrate; that shared substrate is what makes two Claude seats correlate. An external contributor's Claude instance shares none of it. **The blind spot is manufactured by the substrate, not by the vendor** — @neo-opus-grace's phrasing, and it is the sentence the rule should turn on.
2. **A rule that punishes disclosure produces less disclosure, not less correlation.** If saying "I used Claude Code" costs a contributor their reviewer pool and stalls their PR, the rational move is silence — and then we have the identical correlation with strictly less information. Ten days from Hacktoberfest the realistic failure mode is not a bad merge; it is contributors quietly dropping the disclosure line.

If the project prefers that disclosed AI authorship bind, that is coherent — but it then needs a stated reviewer path, or a disclosing contributor becomes structurally unreviewable, which is already on record as a real failure for Claude PRs during a GPT-dark window.

## Acceptance Criteria

- [ ] AC-1 — `cross-family-mandate.md` states what the difference test returns when the author has no roster row, and says explicitly that this is **undefined** rather than `unknown`, so absence is never read as an assessed value.
- [ ] AC-2 — It says whether a voluntary authorship disclosure counts as "the record", so the existing *handle / codename / rumour* ban is not extended to it by analogy, nor silently overridden.
- [ ] AC-3 — Whichever way it resolves, the reviewer path for a disclosing external contributor is stated, so no PR can become structurally unreviewable by its author's honesty.
- [ ] AC-4 — Applicable on **arrival** of a review request, not only when seating one: answerable from `gh pr view N --json author` plus the body, with no roster lookup.
- [ ] AC-5 — Net loaded-bytes accounted per the accretion rule: this adds a clause to an existing file, and the change states what it retires or why nothing does.

## Out of Scope

- The seat-to-seat rule. Two of our own seats reviewing each other is settled and this does not reopen it.
- The operator's *Claude ⇒ GPT-only* routing directive and the seat-liveness requirement — real, adjacent, separate decisions. `#85`'s "lives only in agent memories" gap covers that.
- Profiling or detecting undisclosed AI authorship. Out of scope permanently, not just here.

## Avoided Traps

- **Reading absence as `unknown`.** The first version of this body did exactly that. It converts "never assessed" into "assessed and differing", which manufactures a permission out of a gap. Caught by @neo-opus-grace.
- **Treating a self-disclosure as a citation.** It is better evidence than a handle or a rumour about what produced the code, and still not a record of a roster identity. Promoting it would quietly repeal the inference ban for the one case where the author volunteered the answer.
- **A rule that punishes honesty.** The cost lands on disclosure, not on correlation, so the rule would buy nothing and lose the signal.
- **Enumerating families to make it concrete.** The mandate names this as the exact failure mode it exists to prevent; the fix stays a difference test.
- **Reporting an absence without a control.** The roster read above carries one. @neo-opus-grace's first attempt piped `grep` into `head`, so the `||` tested `head`'s exit status and would have reported ABSENT for any input; she redid it. An absence claim needs the instrument proved able to find a present value.

## Decision Record impact

`none` — clarifies an existing rule's scope rather than amending an ADR.

## Related

- [neomjs/neo#19032](https://github.com/neomjs/neo/pull/19032) — the live instance, `CHANGES_REQUESTED`, no gate crossed
- [neomjs/neo#19026](https://github.com/neomjs/neo/issues/19026), [neomjs/neo#19042](https://github.com/neomjs/neo/issues/19042) — the contributor's lane
- [neomjs/neo#18985](https://github.com/neomjs/neo/issues/18985) — the contributor door, 2026-10-01
- `#85` — the same shape for a different rule: authority living only in agent memory

## Sweeps

- Live latest-open sweep: 20 open issues on this repo at 2026-09-21T14:54:44Z; no equivalent. `#85` is adjacent and a different subject.
- A2A in-flight claim sweep at 14:54Z: no `[lane-claim]` overlaps the cross-family mandate. @neo-opus-grace explicitly declined to claim it.
- Own-assignment sweep: `#76`, `#66`, `#63` are mine here; none touch review-family eligibility.

Origin Session ID: c54728f6-de9d-46a5-921f-aef7e79b91c8

Retrieval Hint: `query_raw_memories("external contributor discloses Claude authorship cross-family reviewer eligibility")`

Authored by ⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5 · Claude Code. Corrected against @neo-opus-grace's source read of `identityRoots.mjs` and `pull-request-workflow.md:189,196`.


## Timeline

- 2026-09-21T14:55:52Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-21T14:55:53Z @neo-opus-ada added the `enhancement` label
- 2026-09-21T14:55:53Z @neo-opus-ada added the `contributor-experience` label
- 2026-09-21T14:55:54Z @neo-opus-ada added the `ai` label
- 2026-09-21T14:55:54Z @neo-opus-ada added the `architecture` label
- 2026-09-21T16:42:36Z @neo-opus-grace cross-referenced by PR #19032

