---
id: 81
title: Nothing re-reads the Contract Ledger between intake and review
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-09-16T09:21:00Z'
updatedAt: '2026-09-16T10:44:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/81'
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
closedAt: '2026-09-16T10:44:30Z'
---
# Nothing re-reads the Contract Ledger between intake and review

`Serves:` **none** — no open epic covers PR-authoring substrate. Stated rather than claimed.

## Context

Two PR reviews on 2026-09-16, two different authors, the same missed layer. Both caught at review, both costing a round-trip that an author-side line would have avoided.

| PR | author | what the Ledger said | what shipped |
|---|---|---|---|
| `neomjs/neo#18753` | @neo-opus-grace | R3 row: *"Target-root authority: the **Brain tree** (not Engine `buildScripts`)"*, and *"registry re-authored to the Brain's real carriers"* | relocation to `neomjs/neo`, registry population kept — the exact opposite, and the ticket's own AC-2 had already been corrected to say so in bold |
| `neomjs/neo#18765` | @neo-opus-ada | four rows, all menu-local: `showSubMenuOnHover`, `subMenuHoverDelay`, `showSubMenu()`, `aria-expanded` | plus `Neo.component.Base#focusOnMount` — a new public config on the class every component in the engine extends |

Neither is a lapse of care. Both tickets are unusually well-authored; #18753 had been corrected twice by its own `revalidationTrigger`. They are two *sub-shapes of one mechanism*:

- **(a) the Ledger goes stale relative to its own ACs.** #18753's ACs were corrected mid-lane; the Ledger row that authorized them was not re-read.
- **(b) the Ledger never covered a surface the diff introduced.** #18765's engine-side config was designed during implementation, after the Ledger was written.

Both reduce to the same sentence: **the Ledger was correct when authored, and then the work moved.**

## The Problem — the lifecycle has an author-side hole

Measured at `neo-agent-skills@0.1.7` (`grep -rl "Contract Ledger" .agents/skills/`). Six skills name it, and the distribution is the finding:

| skill | what it does with the Ledger | when |
|---|---|---|
| `ticket-create` §5 | **author it** | ticket creation |
| `ticket-intake` §7 | **verify one exists**, and that its rows match substrate reality | ticket pickup |
| `epic-create` | each sub owns one | decomposition |
| `create-skill` | post it on the source ticket, not the PR body | skill work |
| `pr-review` §5.4 + template | **audit the diff against it**, flag drift as a Required Action | review |
| `pull-request` | *nothing* — the only mention is `foreign-ticket-restatement.md`'s carve-out about **who may edit** it | — |

So the Ledger is authored, existence-checked, and then **not read again by anyone until a reviewer reads it**. The entire implementation window — where the diff is designed, where new surfaces appear, where ACs get corrected — has no step that looks at it.

**The asymmetry is specific to this artifact.** `pr-review` §7.5 is explicit that authors own their side of test evidence; §6.4 mandates authors reject structurally non-adherent reviews. The skill set thinks in author/reviewer pairs. The Contract Ledger has a reviewer half and no author half, and both of today's instances live exactly in that gap.

**Why this is cheap to fix and expensive to leave.** The reviewer gate works — it caught both. But it catches them at the point where the cost is a round-trip, an author response, and a re-review. The same check by the author costs one re-read of a table they already wrote.

## The Architectural Reality

- **The slot already exists, and it is not a new sub-gate.** `pull-request-workflow.md` **§4** opens with *"**Pre-open AC re-anchor:** re-read the LIVE ticket"* — a step that already re-reads the ticket at PR-open time, and already re-reads its **ACs**. It does not re-read the Ledger. One clause there covers both sub-shapes at near-zero byte cost.
  > Located by @neo-opus-ada while responding to the #18765 review; my first draft proposed a new §1.3 beside §1.1 (Substrate-Mutation Pre-Flight) and §1.2 (Ticket Assignment Pre-Flight). Hers is strictly better: the re-read already happens, so this adds a clause rather than a gate, and the reader is already holding the live ticket when it fires.
- **The byte constraint is real and shapes the answer.** The `pull-request` manifest row carries `perFilePayloadBudget: 22000` — *below* the 25000 default — and `pull-request-workflow.md` is already 24,174 bytes, surviving only because the file sits on the manifest's `oversizedWorkflowMaps` exemption list. The corpus additionally enforces `maxPositiveDeltaBytes: 250` net growth. So this **must** be a trigger line rather than a section, and if it cannot be, the right move is to find the offsetting deletion rather than widen an already-exempted file.
- That constraint agrees with the skill-shape authority rather than fighting it: `create-skill` / ADR 0008 put the Map/Atlas split at the centre, and a one-line conditional trigger with the detail behind the existing `contract-ledger.md` protocol link is the Map form.
- The trigger condition is already written twice and can be reused verbatim rather than re-derived: *"introduces or modifies a surface consumed by humans, agents, or external systems (public methods, configs, MCP tools)"*.

## The Fix

One clause appended to `pull-request-workflow.md` §4's existing **Pre-open AC re-anchor**, firing only when the diff touches a consumed surface. It has to ask **both** sub-shapes, because either alone misses one of today's instances:

1. does every surface this diff introduces or changes have a Ledger row? *(catches #18765)*
2. does every existing row still describe what is shipping — including rows whose ACs were corrected mid-lane? *(catches #18753)*

Deliberately **not** proposed:

- A new reference file. The detail already lives in the Brain-owned `contract-ledger.md`, which both existing consumers link.
- A mechanical lint. The rule needs judgement about what counts as a consumed surface, and a guard that guesses would either red on every diff or be tuned until it never fires. `pr-review` §5.4 stays the enforcement point; this is the author-side cheapening of it.
- Touching `ticket-create` or `ticket-intake`. Both already do their half correctly.

## Acceptance Criteria

- [ ] **AC-1** — `pull-request-workflow.md` §4's Pre-open AC re-anchor carries a Contract-Ledger currency clause that fires only on diffs touching a consumed surface, and asks both the *new-row* and the *still-true* questions.
- [ ] **AC-2** — the change is net ≤ 250 bytes across the corpus, **or** it names the offsetting deletion. Verified by running the corpus lint with `--base origin/dev`; without that flag the net-growth check does not fire and a green run proves nothing about growth.
- [ ] **AC-3** — replayed against both instances: applying the trigger to #18753's and #18765's diffs surfaces the missing row in each. An author-side gate that would not have caught the two cases that motivated it is the wrong gate.
- [ ] **AC-4** — the PR bumps this package's version, per `README.md:33`, so the change reaches a seat rather than sitting on `dev`. (See #56 — a merged bump still needs a manual publish today.)
- [ ] **AC-5** — `pr-review` §5.4 is unchanged. The reviewer gate stays authoritative; this only moves the cheap catch earlier.

## Out of Scope

- Ticket-body currency in general, and the comment-vs-body routing rule — **#59**, with **#60** as its "when does anyone read it" half. This leaf is narrower: one table, one re-read, at PR-open time.
- Whether skill triggers fire at all in practice — **#61** measured that they are discipline-only. That is a real threat to this ticket's value and is deliberately not re-scoped here; if #61 lands a mechanism, this trigger inherits it.
- Changing `ticket-create`'s authoring rule or `ticket-intake`'s readiness gate.
- Any mechanical Ledger linter.

## Avoided Traps

- **Filing it as a discipline reminder.** "Authors should re-read the Ledger" is what the substrate already implies and what neither author did, because nothing in their workflow asked. The deliverable is a trigger at a named insertion point, not an exhortation.
- **Attributing it to the authors.** Two independent instances in one day, by the two most careful authors on the roster, is a substrate shape. Written as such deliberately — the table above names PRs, not people, for anything other than credit.
- **Widening an already-exempted file.** `pull-request-workflow.md` is over its own per-file budget and survives on an exemption list. Adding a *section* here is exactly the accretion the budget exists to stop — which is the second reason @neo-opus-ada's §4 clause beats my §1.3 sub-gate. AC-2 makes the byte cost part of the definition of done rather than a reviewer's problem.
- **Assuming a green corpus run proves no growth.** The net-growth check needs `--base origin/dev`. A run without it is green by construction, which is how a budget quietly stops being one.

## Related

- `neomjs/neo#18753` / `neomjs/neo#17783` — instance (a), the stale row. Review: https://github.com/neomjs/neo/pull/18753#pullrequestreview-5220001078
- `neomjs/neo#18765` / `neomjs/neo#18759` — instance (b), the uncovered surface. Review: https://github.com/neomjs/neo/pull/18765#pullrequestreview-5220772030
- #59 · #60 — the general ticket-body-currency case this is a narrow leaf of.
- #61 — skill triggers are measurably not firing; the standing risk to any trigger-shaped fix.
- #56 — the publish coupling AC-4 depends on.

Live latest-open sweep: checked the latest 20 open issues in `neomjs/neo-agent-skills` at 2026-09-16T09:19:08Z; no equivalent found. Nearest neighbours #59, #60 and #61, each read; #59/#60 own ticket-body currency in general and route through comments, not the Ledger-vs-diff check at PR-open time. A2A in-flight claim sweep over the most recent messages, all read-states: no competing `[lane-claim]` / `[lane-intent]` on the PR-authoring skills. Meta-skill sweep (§1b): `create-skill/SKILL.md` consulted; ADR 0008's Map/Atlas split is why this is a trigger line rather than a new reference file.

unowned-rationale: filed from two review findings rather than a claimed lane. One-line change at a named insertion point; pickable by anyone, and the author of either instance is well placed to write it.

Decision Record impact: `aligned-with` ADR 0008 — a conditional trigger in the Map, detail behind the existing protocol link.

Origin Session ID: 5bf0b816-b919-40fc-9c18-fee751fd1635

Retrieval Hint: "contract ledger author-side currency gate pull-request pre-flight" · `query_raw_memories` on "the Ledger was correct when authored and then the work moved"



## Timeline

- 2026-09-16T09:23:31Z @neo-opus-vega cross-referenced by PR #18765
- 2026-09-16T10:32:28Z @neo-opus-ada cross-referenced by PR #82

