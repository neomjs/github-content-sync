---
id: 57
title: 'The prescription stage never runs at intake, and the carve exempts it on the tickets self-authorship weakens most'
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-09-08T15:30:06Z'
updatedAt: '2026-09-12T11:14:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/57'
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
closedAt: '2026-09-12T11:14:40Z'
---
# The prescription stage never runs at intake, and the carve exempts it on the tickets self-authorship weakens most

`Serves:` **none** — the one open epic (#14) is PR governance; this is ticket lifecycle. Stated rather than claimed.

> **Amended 2026-09-09 after @neo-gpt's review of #58.** The original body claimed intake asks *nothing* about the prescription. That is false, and the correction is below in place: intake §1 step 10 asks it. The surviving gap is narrower — the routes that skip §1. My own Revalidation trigger fired here, so the test it names is applied at the bottom rather than quietly dropped.

## The gap

The six-stage challenge chain exists and stage 2 is exactly the check this ticket is about:

> `ticket-create` §2.2 — **Prescription** — is the stated fix the right substrate for the problem, or does it treat a symptom? Could a different layer (config, service, daemon, schema) solve it better?

It runs at **creation** (`ticket-create §2`), retrospectively at **triage** (`ticket-triage` Step 1, for unlabeled tickets), and — this is the part the first draft of this body got wrong — **at intake too**, as §1 step 10:

> **Hypothesis vs. Root Cause Validation:** Tickets frequently prescribe specific technical solutions (e.g., "Implement X to fix Y"). You MUST NOT accept the prescribed solution blindly. You must independently investigate the systemic behavior to verify if 'X' is actually the correct solution for 'Y'.

So the challenge is not missing from intake. **It is missing from the routes that skip intake §1.**

| intake route | reaches step 10? |
|---|---|
| full gate — another agent authored the ticket | **yes**, and this is the majority case |
| Hot Context Fast-Path — you created it this session | **no** — *"generally exempt from the Validation Sweep (Section 1)"* |
| self-authored carve — you authored it, any session | **no** — `SKILL.md` routes to the carve before the workflow is read at all |

Step 10 lives inside `## 1. The Validation Sweep`. Everything that exempts §1 exempts it, and both exempting routes are keyed on **self-authorship** — which is the one condition under which a prescription challenge is least independent, not most.

## The carve's rationale inverts on stage 2

`references/self-authored-carve.md`:

> | Artifact **and** reasoning — **you authored it this session** | **Exempt.** `ticket-create`'s six-stage chain ran in this same context window. |

That is sound for stages 1 and 3–6, which test facts about the codebase — the same context window really did establish them. It is **backwards for stage 2**. Stage 2 challenges the prescription *the author wrote*; the chain having run "in this same context window" means it was run **by the person whose prescription it is**, in the context that produced it. Same-session self-authorship is the condition that removes stage 2's independence, and the carve reads it as the credential that supplies it.

## Evidence

neomjs/neo#18460 → neomjs/neo#18473, closed after three review cycles. Authored and taken same-session by @neo-opus-vega, so exempt. Its body carries the disposition in her own words:

> **Disposition: ticket-prescription-off.** I authored the earlier instruction to create a separate `VesselController` beside `WorkspaceController`. That selected a superclass and file before establishing the responsibility's [owner].

The premise was real (vessel policy needs a lifetime owner) and the ticket was hours old, so §0 and §1.3 both pass. `src/controller/Component.mjs` was one open away the whole time — and note what that anchor actually shows: the file was available to read, so what survived three cycles was not an unread file but an unasked question. A read receipt would not have caught it; the comparison would.

Cost: a day of hers, three review cycles of @neo-gpt-emmy's.

## The second half: intake accepts or rejects, it never sharpens

@tobiu:

> *challenging if a ticket is about "building the right thing" => which requires to explore best practices briefly (reading guides, memories, related code files). so tickets are crystal clear, most are not.*

Intake is a binary gate: accept, or run the Rejection Protocol. There is no arm for the common case — the ticket is right and **unclear**. The taker has just done the exploration and holds the hottest context anyone will have on it, and there is no instruction to write that back. Every later reader re-derives it, including the reviewer who reads the ticket to judge the PR.

This half is untouched by step 10 and stands as originally filed.

## AC

- **AC-1** The prescription challenge survives the routes that skip §1 — the Hot Context Fast-Path and the self-authored carve. It invokes the existing `ticket-create §2` chain rather than restating it, and §0 is where it lives because §0 already sits outside the fast-path. No new mechanism, and step 10 stays the authority on the full-gate route.
- **AC-2** `self-authored-carve.md` stops exempting stage 2, and says why self-authorship is the trigger rather than the credential for that one stage. Stages 1 and 3–6 keep the carve.
- **AC-3** Intake gains a third outcome between accept and reject: sharpen the ticket from what the exploration established. Routed by authorship per `ticket-create` §11 — your own body in place, someone else's by comment. Bounded: one edit or one comment, not a re-authoring.
- **AC-4** *(amended — the original said the added clauses would be offset by tightening §0, "measured in the PR". Measured, and the offset is real but does not cover the addition. The honest accounting and its load-path assessment now live here rather than only in a PR body.)*

  Against merge-base `f5fcb7f1`:

  | exposure | delta | loaded when |
  |---|---|---|
  | `skills.manifest.json` description | +30 | skill index — **every turn** |
  | `ticket-intake/SKILL.md` body | +165 | on **skill activation**, not every turn |
  | `references/self-authored-carve.md` | +1,903 | self-authored route |
  | `references/ticket-intake-workflow.md` | +1,561 | full-gate route — **and** the self-authored route when the drift probe is non-empty |
  | **total** | **+3,659** | |

  Three exposures, not one: **every turn +30** (only the manifest description sits in the index — `SKILL.md` is charged on activation, not because it changed), **per activation +195**, and **worst-case route +3,659**, the whole total. A self-authored ticket whose drift probe returns non-empty reads the carve and is then sent by it to the full workflow, so that route loads both references — the carve reduces expected load, it does not cap it.

  The carve's growth is the one to justify rather than excuse: it is the file whose purpose is to let a ~4.6 KB read retire a 24.5 KB one, so +1,903 there is net-negative in expectation on the route that loads it, and pays nothing on the route that does not.

  The §0 tightening promised in the original AC was delivered — its intent paragraph restated `ticket-create` §0 near-verbatim and now references it — but recovers only ~108 bytes. **This PR does not net-reduce loaded bytes, and I am not going to present it as though it does.** Retirement trigger, per the accretion rule: when a corpus lint can assert the `Prescription checked:` receipt names a decision rather than only a path, the prose arguing for that decision can compress to the lint's error message.

## Revalidation trigger — fired, and applied

> *If a reader shows stage 2 is already reachable at intake through a path this body missed, close as `already-resolved` — the finding is an absence, and absences are the claims most worth falsifying.*

@neo-gpt did exactly that, and the trigger is honoured rather than reinterpreted: **AC-1 as originally worded is already-resolved** on the full-gate route, and its wording above is narrowed to what remains true. The ticket does not close, because AC-2 (the carve's stage-2 exemption) and AC-3 (the third outcome) are untouched by step 10 — both exempting routes still reach neither. An absence claim that survives as "on two of three routes" is a smaller finding than the one filed, and it is the one this ticket now carries.


## Timeline

- 2026-09-08T15:30:07Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-08T15:30:08Z @neo-opus-grace added the `bug` label
- 2026-09-08T15:30:08Z @neo-opus-grace added the `ai` label
### @neo-opus-grace - 2026-09-08T15:35:46Z

**AC-4 amended — I claimed an offset I did not deliver.**

Filed text: *"the added clauses are offset by tightening intake §0."* Measured on the branch:

```
ticket-intake-workflow.md   22875 -> 23968   (+1093)
self-authored-carve.md       2726 ->  3797   (+1071)
                                     net     +2164
```

I removed one redundancy — §1's `Relevance Validation` step, whose question §0 now asks more generally, and which also collapsed a duplicate `4.` numbering — but the two new clauses are load-bearing and I am not going to shrink them into uselessness to satisfy a number I wrote.

**AC-4 as it should read:** the Accretion Defense permits net-growth against a cited retirement trigger, and this is the trigger — `Prescription checked: <path>` is a PR-body anchor. When `check-pr-body.mjs` enforces it, the §0 prose collapses to a pointer at the lint, and the carve's three-step list retires with it. That is the same shape this repo already uses: prose holds the rule until a check can, then the check owns it and the prose goes.

Both files are references, loaded on skill trigger rather than per turn, and `check-substrate-size.mjs` and `lint-skill-corpus.mjs` both pass on the branch.


- 2026-09-08T15:36:41Z @neo-opus-grace cross-referenced by PR #58
- 2026-09-08T15:37:51Z @neo-opus-grace referenced in commit `ab133b7` - "refactor(ai): intake challenges the prescription, and stage 2 loses the self-authored exemption (#57)

The six-stage chain runs at creation and retrospectively at triage. It never ran
at intake — the moment immediately before someone writes code. Intake's §1.3
covers stage 1 only and §0 asked about staleness, so a ticket could be fresh, its
problem real, and its prescribed fix wrong, with nothing positioned to catch it.

The carve's exemption inverts on stage 2 specifically. That stage challenges the
fix the author chose, so "the chain ran in this same context window" means it was
run by the author of the thing under challenge. Same-session authorship removes
stage 2's independence rather than supplying it.

Both clauses discharge as a read with a citable artifact rather than as a
judgment, per the carve's own constraint that a gate you can talk yourself out of
is not a gate. The §0 placement is deliberate: §0 sits outside the Hot Context
Fast-Path, so the check survives the exemption it corrects.

Intake also gains a third outcome. It could accept or reject; most tickets are
neither wrong nor clear, and the taker holds the hottest context anyone will have
on that ticket.

Removes §1's Relevance Validation step, whose question §0 now asks more generally.
That also collapses a duplicate `4.` list numbering.

[skill-growth-justified: +2164 bytes across two reference files, loaded on skill
trigger rather than per turn. The clauses are gates, not prose, and trimming them
to fit 250 bytes would leave the gate unrunnable; one redundancy was removed
rather than none. Retirement trigger, per the Accretion Defense: `Prescription
checked: <path>` is a PR-body anchor, so when check-pr-body.mjs enforces it the
§0 prose collapses to a pointer and the carve's three-step list retires with it.]"
- 2026-09-08T15:38:28Z @neo-opus-grace referenced in commit `6210621` - "refactor(ai): intake challenges the prescription, and stage 2 loses the self-authored exemption (#57)

The six-stage chain runs at creation and retrospectively at triage. It never ran
at intake — the moment immediately before someone writes code. Intake's §1.3
covers stage 1 only and §0 asked about staleness, so a ticket could be fresh, its
problem real, and its prescribed fix wrong, with nothing positioned to catch it.

The carve's exemption inverts on stage 2 specifically. That stage challenges the
fix the author chose, so "the chain ran in this same context window" means it was
run by the author of the thing under challenge. Same-session authorship removes
stage 2's independence rather than supplying it.

Both clauses discharge as a read with a citable artifact rather than as a
judgment, per the carve's own constraint that a gate you can talk yourself out of
is not a gate. The §0 placement is deliberate: §0 sits outside the Hot Context
Fast-Path, so the check survives the exemption it corrects.

Intake also gains a third outcome. It could accept or reject; most tickets are
neither wrong nor clear, and the taker holds the hottest context anyone will have
on that ticket.

Removes §1's Relevance Validation step, whose question §0 now asks more generally.
That also collapses a duplicate `4.` list numbering.

[skill-growth-justified: +2164 bytes across two trigger-loaded reference files; the added clauses are gates whose steps cannot be trimmed to 250 bytes and still run, and one redundancy was removed rather than none. Retirement trigger: `Prescription checked: <path>` is a PR-body anchor, so when check-pr-body.mjs enforces it the §0 prose collapses to a pointer and the carve's three-step list retires with it. Detail in #57.]"
- 2026-09-08T22:16:19Z @neo-opus-grace referenced in commit `946cd60` - "refactor(ai): the activation surface says the prescription check survives the exemption (#57)

The frontmatter description is what a harness loads to decide whether to open the
skill at all, and it said "A ticket YOU authored this session is exempt" without
qualification. The amended carve says the opposite for stage 2. An agent reading only
the trigger — which is the common case, since the description exists to save the read —
would take the exemption whole and never reach the clause that narrows it.

Found by @neo-gpt reviewing the §0 placement: §0 does survive the Hot Context Fast-Path,
but a self-authored intake never arrives there, because SKILL.md routes it to the carve
first and zero drift permits skipping the payload. The carve is therefore the reachable
enforcement on that route, and the trigger was still advertising the unnarrowed rule."
- 2026-09-08T22:16:57Z @neo-opus-grace referenced in commit `04f1b68` - "refactor(ai): the activation surface says the prescription check survives the exemption (#57)

The frontmatter description is what a harness loads to decide whether to open the skill
at all, and it said "A ticket YOU authored this session is exempt" without qualification.
The amended carve says the opposite for stage 2. An agent reading only the trigger —
the common case, since the description exists to save the read — would take the exemption
whole and never reach the clause that narrows it.

The manifest row carries the same description, and the corpus lint refuses when the two
disagree: the router and the index would advertise different triggers. Both are updated
together, which is the point of that check.

Found by @neo-gpt reviewing the §0 placement: §0 does survive the Hot Context Fast-Path,
but a self-authored intake never arrives there, because SKILL.md routes it to the carve
first and zero drift permits skipping the payload. The carve is therefore the reachable
enforcement on that route, and the trigger was still advertising the unnarrowed rule.

[skill-growth-justified: +180 bytes across a frontmatter description, its manifest row and one SKILL.md line. The growth IS the fix — an activation surface that advertises an exemption the payload narrows is the defect, and narrowing it costs a clause. Retirement trigger unchanged from #57: when check-pr-body.mjs enforces the `Prescription checked:` anchor, the prose collapses to a pointer.]"
- 2026-09-08T22:48:10Z @neo-opus-grace referenced in commit `eea4fd5` - "refactor(ai): stage 2 discharges as the comparison, not the read (#57)

Euclid's R1 on #58 found three contract gaps.

RA-1: the carve's name/open/cite discharge proved a file was opened, which
is not what fails stage 2. Its own anchor shows why — #18460's prescribed
class was one open away through three review cycles. The read becomes the
prerequisite; the discharge is stage 2's canonical question, answered in the
record as 'owns the concern' or 'better owner: <path>'. The failure-mode
section is reconciled with it: a judgment that lets you skip is the loophole,
a judgment the gate obliges you to write down is the gate.

RA-2: 'accept and sharpen' routes by authorship per ticket-create §11 —
your own body in place, someone else's by comment.

RA-3 (partial): §0 no longer claims nothing at intake challenges the
prescription. §1 step 10 does; it sits inside the Validation Sweep, which
the fast-path and the carve both skip. §0's intent paragraph restated
ticket-create §0 near-verbatim and now references it."
- 2026-09-12T11:14:40Z @tobiu referenced in commit `317b3b1` - "Merge pull request #58 from neomjs/grace/57-intake-prescription-stage

refactor(ai): intake challenges the prescription, and stage 2 loses the self-authored exemption (#57)"
- 2026-09-12T11:14:40Z @tobiu closed this issue

