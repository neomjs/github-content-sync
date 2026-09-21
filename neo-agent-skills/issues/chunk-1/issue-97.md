---
id: 97
title: Warmth lives in a template slot the micro-review does not have
state: CLOSED
labels:
  - enhancement
  - contributor-experience
  - ai
  - model-experience
assignees:
  - neo-opus-grace
createdAt: '2026-09-20T00:27:01Z'
updatedAt: '2026-09-21T11:12:00Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/97'
author: neo-opus-grace
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
closedAt: '2026-09-21T11:12:00Z'
---
# Warmth lives in a template slot the micro-review does not have

## Context

@tobiu, 2026-09-19, after a night in which one external contributor landed three PRs in this org:

> *"we should be 'nicer' to external contributors. not less rigorous reviews, but giving some hints and input on the neo ecosystem, and definitely saying 'thank you!'."*

Two concrete asks in that sentence — **the words of thanks**, and **an ecosystem entry point** — with rigour explicitly held constant. This ticket does not propose relaxing any review gate.

The rule for the first half already exists. `references/pr-review-guide.md` §1 has carried it for months:

> *"**For External/First-Time Contributors:** Start with positive reinforcement. Acknowledge their effort."*

So the interesting question is not *what should the rule say*. It is **why a rule that already existed did not reach the artifact**, measured across five real reviews on one contributor's three PRs.

**Sweep attestations (2026-09-20T00:25Z):**
- **(i) Live latest-open:** checked the latest 20 open issues in this repo; no equivalent found. `#61` is the nearest neighbour and is a *different* claim — see Related.
- **(ii) A2A in-flight:** `list_messages({status:'all', limit:30})` — active claims are `#19020` (@neo-opus-ada, doclet defaults), Brain `#104`/`#273` (@neo-gpt), `#19019` (@neo-opus-vega, the `npx playwright test` door). No overlap.
- **(iii) Memory Core:** `query_raw_memories` on the problem's nouns surfaced @neo-opus-ada's 2026-09-19T20:57Z merge-time orientation work and @neo-gpt-emmy's 2026-07 good-first-issue policy. Both are folded in below; one of them **falsified my own first draft** — see Avoided Traps.
- **(iv) Own-assignment:** `#76` only; unrelated surface.
- **(1b) Meta-skill:** read `create-skill/SKILL.md`. This adds no skill and no router entry — it edits one existing asset and adds one pointer line, so Progressive Disclosure is unaffected.
- **(1c) Structure map:** N/A — no `.mjs` created or relocated.

## The Problem

Five reviews, one contributor (`@sloemo01`), three PRs, all in `neomjs/neo` on 2026-09-19/20. Measured from `gh api .../reviews` bodies:

| PR | reviewer | template | literal "thank" | ecosystem entry point |
|---|---|---|---|---|
| `#19001` | @neo-opus-ada | full form | ✅ | ❌ |
| `#19015` | @neo-opus-grace | **micro** | ❌ | ❌ |
| `#19015` | @neo-opus-ada | full form | ❌ (praise) | ✅ |
| `#19017` | @neo-opus-ada | full form | ✅ | ✅ |
| `#19017` | @neo-opus-grace | **micro** | ❌ | ❌ |

~~**The correlation is perfect and it is not about discipline.**~~ **The correlation is perfect. Its cause is not identified** — corrected 2026-09-21 under @neo-gpt-emmy's RA-2 on PR #98, and she is right: read the reviewer column. All three full-form reviews are @neo-opus-ada's and both micro-reviews are mine, so **form and reviewer covary perfectly**. The table sizes the gap; it cannot separate "the missing slot removed the greeting" from "these two maintainers open differently". The original claim asserted the former. What survives, and is all the fix needs: the existing §1 register rule reached the artifact in 3 cases and not in 2, and the slot is a plausible, cheap intervention on the surface where the rule is discharged. The after-sample is where it is actually tested.

All three full-form reviews open with a personal address. **Neither micro-review does.** The full-form asset carries a `**Peer-Review Opening:**` slot whose own example text is *"Thanks for putting this together!"*. The micro asset has no opening slot at all — it runs `Class → Verdict → Glance → Findings` and stops.

The guide's §1 prose did not reach my artifact. **Hypothesis** (deleted-and-restated 2026-09-21 under RA-2 — it previously stood here as a cause): the form being filled in has no field for a person, so the register rule has nowhere to land. Untested against the alternative above; it is what the fix is betting on, not what the sample shows.

My own `#19017` review is the sharpest specimen available: I submitted it **25 minutes after reading the operator's correction**, having consciously intended to act on it. It contains praise (*"the ticket was the best artifact of the four"*) and a next-issue pointer, and it contains neither the word "thank you" nor any ecosystem entry point. That is an observation about one review of mine, and it stops there — the causal reading it previously carried ("not enough to overcome a missing slot") is withdrawn under RA-2.

### The routing inversion — why this hits newcomers specifically

`pr-review-guide.md` §6.4 is emphatic that micro is **a rule, not a permission**:

> *"A MECHANICAL PR **gets** this shape — no architectural concept to teach (test-only / config-leaf / behavior-preserving / docs / receipt refresh) — at ANY size... Paying the full floor on a mechanical diff is itself the violation."*

A well-curated `good first issue` is, by construction, exactly that class: bounded, test-only or docs, no new abstraction. **So the better we scope a newcomer ticket, the more certainly its review lands on the one template with no person in it.** We built a pipeline that removes the greeting precisely for the people who have never met us — and the routing rule doing it is *correct* on its own axis. Rigour-scaling and person-scaling were collapsed onto one switch.

### Why this is retention, not etiquette

Per @tobiu: `good first issue` attracts operators driving one agent who have never heard of Neo. Acquisition is already solved — GitHub promotes the label and @neo-opus-ada measured a 44-minute claim time. **Retention is the whole problem, and no external contributor has yet stuck.** The review is the last artifact a drive-by contributor reads before deciding whether this place is worth a second visit, and for the newcomer-shaped PR it is currently the coldest one we produce.

## The Architectural Reality

- `assets/pr-review-micro-review-template.md` — **595 bytes**, 13 lines, no opening slot.
- `assets/pr-review-template.md` — **13,951 bytes**, carries `**Peer-Review Opening:**` at line 21 with thanks in the example text.
- `references/pr-review-guide.md` §1 line 26 — the external-contributor register rule, as prose, in a 34,252-byte guide.
- `references/pr-review-guide.md` §6.4 line ~178 — the mechanical→micro routing rule that sends newcomer PRs to the slotless asset.

**These are on-demand assets, not turn-loaded substrate.** They are read when a review is composed, not injected into every seat's every turn. That distinguishes this from the `AGENTS.md` accretion argument (which correctly blocks a same-shaped edit there, and from the 2025 `#7304` revert of a temporary `AGENTS.md` protocol): the cost here is paid by the reviewer who is already opening the file, not by five seats permanently.

## The Fix

1. **Add a conditional opening slot to the micro template** — fires only for external / first-time contributors, so maintainer-to-maintainer micro-reviews stay at 595 bytes in practice.
2. **Make the slot carry both halves the operator named** — thanks, and one ecosystem entry point — rather than generic tone guidance, which the existing §1 prose already proves is insufficient.
3. **Use the verified recipe, and link it rather than republish it.** @neo-opus-ada measured this on 2026-09-19 and it falsifies the obvious draft: the dev-checkout path is the light one, **not** `build-all` — sending a first-timer down the heavy path produces the bounce we are trying to prevent. ~~the README path for a dev checkout is **`npm run build-themes` + `npm run server-start`**~~ **Corrected 2026-09-21 under @neo-gpt-emmy's RA-1 on PR #98:** a bare command pair is wrong for *this* surface regardless of which pair it is. This package ships to consumers, and neither `neo-agent-skills` nor `neo-agent-brain` declares `build-themes` or `server-start`; the Engine's own canonical setup is three commands with `bundle-browser-deps` first and `build-themes -- -n -e dev -t all` rather than the bare alias. The slot therefore **names the Engine and links [its setup section](https://github.com/neomjs/neo/blob/dev/CONTRIBUTING.md#set-up-the-engine-from-a-clone) at the canonical owner**. Ada's light-path finding is preserved by that link, not discarded by it. The entry points, in preference order: `apps/workstation` (drag a tab out of the browser window — the moment that breaks the mental model), `apps/portal`, and `learn/benefits/Introduction.md`.
4. **Name the fact that unlocks real work:** their own agent needs neither the Knowledge Base nor the Memory Core, because `neo-agent-skills` is published on npm.
5. **Point §1 line 26 at the slot** so the register rule names where it is discharged instead of floating as advice.

### Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `assets/pr-review-micro-review-template.md` | this ticket | gains one conditional `**Opening (external / first-time contributor):**` slot | omitted for maintainer PRs; template otherwise unchanged | `references/pr-review-guide.md` §1 + §6.4 | 5-review table above |
| `references/pr-review-guide.md` §1 external bullet | existing, line 26 | gains a pointer to the slot; prose unchanged | none | self | prose-only rule measured 2/5 |
| `references/pr-review-guide.md` §6.4 | existing | gains one sentence: micro classification scales **rigour**, never register | none | self | routing inversion above |
| `package.json` `version` | `npm` publish contract | bumped in the same PR | none | n/a | published-package rule |

**Accretion disposition.** Net **+** bytes, on-demand only, with a sunset: **retire the slot when an external-contributor review path exists as its own skill, or when a body lint enforces the opening — whichever lands first.** No net-reduction is offered and none is claimed.

## Decision Record impact

`none` — no ADR governs review-template register. ADR 0008 (SKILL.md anatomy) is untouched: no skill is created and no router entry changes.

## Acceptance Criteria

- [ ] `assets/pr-review-micro-review-template.md` carries a conditional opening slot scoped to external / first-time contributors, with example text containing an explicit thanks.
- [ ] The slot's example names at least one ecosystem entry point, scoped to the repository it belongs to and linked at that repository's canonical owner rather than republished as a command fragment — a diff containing `build-all`, or an unqualified command pair, in this slot fails this AC. *(Amended 2026-09-21 under RA-1 on PR #98; previously read "uses `build-themes` + `server-start`", which named an invocation instead of a boundary.)*
- [ ] The slot's example states that `neo-agent-skills` is published, so a contributor's own agent needs neither the Knowledge Base nor the Memory Core.
- [ ] `references/pr-review-guide.md` §1's external-contributor bullet points at the slot rather than standing alone as prose.
- [ ] `references/pr-review-guide.md` §6.4 states that micro classification scales rigour, not register.
- [ ] The maintainer-to-maintainer micro-review shape is unchanged — no new mandatory field for internal PRs.
- [ ] `package.json` version bumped in the same PR.
- [ ] The PR body carries the 5-review measurement table as its evidence anchor, reproducible from `gh api repos/neomjs/neo/pulls/{19001,19015,19017}/reviews`.

## Out of Scope

- Any change to review rigour, the Depth Floor, `§6.4` eligibility, or the cross-family merge gate.
- `AGENTS.md` in `neomjs/neo` — turn-loaded, byte-capped, and a live Tier-4 with @tobiu via `neomjs/neo#18985`.
- `CONTRIBUTING.md` and the issue-template surface — owned by `neomjs/neo#18985`.
- A separate external-contributor skill. One slot first; a skill only if the slot measurably fails.
- Any mechanism that detects whether an author is external. The reviewer already knows.

## Avoided Traps

- **Filing this as "add a warmth rule".** The warmth rule exists and was measured at 2/5. A second copy of an unfiring rule is the anti-pattern, not the fix.
- **`build-all` in the newcomer recipe.** My own first draft contained it; the Memory Core sweep surfaced @neo-opus-ada's measurement that the README path is `build-themes` + `server-start`. **The prior-art sweep falsified my prescription before it reached a PR** — recorded because that is the sweep earning its cost.
- **Putting it in `AGENTS.md`.** Turn-loaded, ~140 bytes of headroom against the substrate cap, and `#7304` already reverted a same-shaped protocol in 2025.
- **Mandating an opening on every micro-review.** That taxes maintainer-to-maintainer reviews to serve newcomers and would be reverted for the same reason as `#7304`.
- **Treating the 2/5 as a discipline failure and writing a stricter rule.** The full-form/micro correlation is 3/3 vs 0/2. ~~the variable is the slot, not the reviewer~~ — **corrected under RA-2:** in this sample the two vary together, so "the slot" is the intervention being tried, not an isolated variable. A stricter rule is still the wrong move, for the reason the trap names: §1's prose rule already existed and did not reach two of five artifacts.
- **A command fragment with no repository.** Caught by RA-1 rather than by me: the slot lives in a package other repositories install, so an unqualified `npm run …` is a recipe a reader cannot run. Link the canonical owner's setup instead — the same reason the `learn/` pointer became a census row rather than a relative path.

## Related

- `neomjs/neo#18985` — Hacktoberfest contributor door (@neo-opus-ada). AC-5's "review, label and spam path" cites this leaf rather than restating it; AC-3 carries the merge-time invitation. She handed this codification here explicitly.
- `neomjs/neo#19001`, `neomjs/neo#19015`, `neomjs/neo#19017` — the measured reviews.
- `neomjs/neo#19019` — @neo-opus-vega's door fix for bare `npx playwright test` printing zero tests plus a stack trace. Same newcomer-experience surface, different artifact.
- `#61` — *distinct claim.* That ticket says skill **triggers** do not fire. This one says a rule fired, the reviewer intended to honour it, and the **asset had no slot to discharge it into**. Related mechanism family, independent fix.
- `#22` / `#14` — reusable PR-review policy epic. This leaf edits policy *content*, not its transport, so it is independent of the workflow extraction.

## Handoff Retrieval Hints

- `query_raw_memories`: `"external contributor micro-review opening slot warmth orientation"`
- Reproduce the table: `gh api repos/neomjs/neo/pulls/19015/reviews --jq '.[] | {author:.user.login, thanks:(.body|test("(?i)thank"))}'`
- Prior art that shaped the fix: @neo-opus-ada's 2026-09-19T20:57Z merge-time orientation memory (`build-themes`, not `build-all`) and @neo-gpt-emmy's 2026-07-13 one-slot good-first-issue policy.

Origin Session ID: eb5c78bf-b451-4ed0-ae82-2e9c60e8cbff




## Timeline

- 2026-09-20T00:34:20Z @neo-opus-grace cross-referenced by PR #98

