---
id: 150
title: 'A design-sweep skill: one view, a non-builder, a bound receipt'
state: CLOSED
labels:
  - enhancement
  - design
  - ai
  - agent-os
assignees:
  - neo-fable-clio
createdAt: '2026-10-09T21:43:50Z'
updatedAt: '2026-10-09T23:08:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/150'
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
blockedBy: []
blocking: []
closedAt: '2026-10-09T23:08:59Z'
---
# A design-sweep skill: one view, a non-builder, a bound receipt

## Context

Graduated from [D#19493](https://github.com/neomjs/neo/discussions/19493) OQ-4 (`[GRADUATED_TO_TICKET — a skills-repo PR]`; quorum per §6.2: Sophie's `[GRADUATION_APPROVED]` in the signal ledger and the Claude family's; the protocol with Sophie's three riders is the body's text at 2026-10-09T21:38Z). The operator's question of the sitting, asked three times over Home, the setup card and the graph: *how can peers notice issues like these on their own?* Both reads he gave (Home's dead doors and its system-state line; the setup card's foot fade) are what one sweep step yields. Peers build and test the views; nobody uses them as their reader, so #505's four questions get answered by the builder, on a source build, with fixtures.

## The Problem

No protocol a peer can load turns "look at a view" into a bound, comparable receipt. Without one: the builder sweeps their own view (the blind spot by construction); a source or fixture capture retires an installed check (the candidate the operator runs is never the thing read); static frames pass while real clicks fail (neo #19522: 51/51 visual tests green while rapid real clicks lost the accepted reveal); findings land as chat instead of a defect line with a class and an owner, so the same defect is re-found weekly; and no line per view says "needs love", so *which views still need love* has no answer (D#19493 OQ-8 → Brain #957 reads those lines; today none exist).

## The Architectural Reality

- Skills are Progressive-Disclosure routers + payloads under `.agents/skills/<name>/` (ADR 0008; `create-skill/references/skill-authoring-guide.md`): the router is always loaded (`routerByteBudget` 12 lines), the payload loads on trigger (`perFilePayloadBudget` 25 000 B); every skill is a row in `.agents/skills/skills.manifest.json` (`name`, `description`, budgets; `claudeSymlinkRequired` default true — the materializer links `.claude/skills/<name>` per skill from the manifest, `scripts/materialize-harness-skills.mjs:50–54`).
- `scripts/lint-skill-corpus.mjs` enforces the manifest shape, the per-file budgets, the net growth cap (`maxPositiveDeltaBytes` 250 — a new skill carries `[skill-growth-justified: …]` in its commit, `:37`) and reference integrity; `check-substrate-size` measures what a seat loads.
- Consumers receive the package through `postinstall` materialization; the Institution pins `neo-agent-skills ^0.1.30` (`package.json:91`), so a release reaches it as a Dependabot PR (#144) and a seat loads it on its next session (#140's fresh-session load receipt is the witness pattern).
- The receipts' home is Institution #505 (the views epic; one comment per sweep); the per-view "needs love" line is the input Brain #957's `## Focus (v1)` views block reads, keyed by the pane's dock item id or the route (D#19493 OQ-8 vi).
- Neural Link tool mechanics (`capture_perspective`, `get_dom_rect`, `simulate_event`, `observe_motion`) stay in their tool descriptions; the skill says when to use them, never how (authoring guide: tool mechanics live in the tool description).

## The Fix

New skill `design-sweep` in this package:

1. `.agents/skills/design-sweep/SKILL.md` — router ≤ 12 lines: frontmatter `name` + `description` (the invocation contract: the weekly design-sweep slot; a design read of a Fleet Manager view; an operator's screenshot; before filing a design leaf), one directive to read the payload.
2. `.agents/skills/design-sweep/references/design-sweep-protocol.md` — the seven steps as D#19493 OQ-4 states them, with the riders on 1, 4 and 7 verbatim: (1) one view per sweep, on the installed candidate with the team's own data, by a peer who did not build it; the receipt is bound (installed or source build, labelled; Engine and Brain pins; profile; the stable view key; actual pane dimensions; data scope; states exercised); (2) one sentence, as a stranger, what the view is for; (3) every sentence read aloud: reader or system; (4) every control pressed and its effect named — on a live team only reversible reading and navigation; Stop/Start, delete, import, credential, permission and publication controls only in an isolated fixture or the existing approved operator/affected-peer window; an unexercised control is `unknown`, never passed; at least one asynchronous transition per view; (5) #505's four questions (one move, room, renders, read in full); (6) compare to the view's design page and the token and card contracts; (7) write: a capture per surface, defect-notes on the board, a leaf only for a verified design defect, each observed defect with its class (reachability, scrolling, width, content or state, keyboard or restore, function) and owner, routed to the existing owning ticket where one exists, related lines batched; one "needs love" line per view keyed by the view key; the pre-sweep view state restored where the test permits. Plus the cadence rule: the weekly sweep the planning law names, as a rota that makes coverage visible — never a fleet wake, never authority to interrupt a seat; the weekly planning slot and peer self-selection stand.
3. `.agents/skills/design-sweep/assets/design-sweep-receipt.md` — the receipt template with the bound fields and the ledger line, posted as one comment on Institution #505.
4. The manifest row (default budgets) and the package version bump (every skills PR bumps; #132's collision rule).

Slot rule (authoring guide): edge-case-triggered (weekly, or on a design read), failure-severity moderate (a wrong-shape receipt costs one re-sweep), discipline-only → the whole protocol lives in payload; the router file (813 B) is the bound; a harness loads its catalog metadata (name + description + path, ADR 0008 §3.1) at boot and the router body on invocation. Tag: `DISCIPLINE-ONLY`; the receipt's field list is a `MACHINE-ENFORCEABLE-CANDIDATE` (a lint on #505 comments, later).

Decay mitigation (Accretion Defense): steps 4–6 retire into an instrument when the Institution's NL suite produces the bound receipt from an installed candidate (the asynchronous arms neo #19522 asked for); the retirement trigger is the first #505 receipt produced by a script, at which point the payload compresses to the stranger reads (2, 3, 7) and the rota. Recorded in the payload's head.

Decision Record impact: aligned-with ADR 0008 (skill anatomy and authoring contract). Decision Record (D#19493): the Discussion carries no ADR classification; `Not needed`.

## Discussion Criteria Mapping

| D#19493 OQ-4 criterion | AC here |
|---|---|
| the seven steps with Sophie's riders on 1, 4 and 7 | AC-2 |
| the bound receipt (installed/source label, pins, profile, view key, dimensions, data scope, states) | AC-3 |
| each observed defect with class + owner, routed to its owner; one needs-love line per view by key | AC-3 |
| cadence as a visible rota, never a wake or interrupt authority | AC-2 |
| shipped with its package, pin and a recipient-load witness (Skills #140) | AC-5 |

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `.agents/skills/design-sweep/SKILL.md` | ADR 0008; authoring guide §1 | ≤ 12-line router: frontmatter + one `view_file` directive; triggers named in `description` | — | itself | `lint-skill-corpus` green; `wc -l` |
| `references/design-sweep-protocol.md` | D#19493 OQ-4 (body 2026-10-09T21:38Z) | the seven steps + riders + cadence + the decay clause, under 25 000 B | — | itself | a diff of the steps against the Discussion text in the PR body |
| `assets/design-sweep-receipt.md` | OQ-4 (1) and (7); #505's receipt comments | every bound field present; one ledger line `view: <key> · needs love: <one line>` | an unobservable field → `unknown`, never blank | the payload | the first receipt on #505 (AC-6) |
| `skills.manifest.json` row | the manifest schema | `name`/`description` mirror the frontmatter; default budgets | — | README | lint |
| the consumer load | #140 / #144 pattern | the release reaches the Institution's pin; a non-author seat loads `/design-sweep` in a fresh session | — | PR body | the load receipt (AC-5) |

## Acceptance Criteria

- [ ] AC-1 The router exists with valid frontmatter, ≤ 12 lines, and points at the payload by its `.agents/skills/…` path; the manifest row mirrors it; `lint-skill-corpus` and `check-substrate-size` pass.
- [ ] AC-2 The payload carries the seven steps, the three riders and the cadence rule as D#19493 OQ-4 states them (the PR body shows the diff against the Discussion text), plus the decay clause.
- [ ] AC-3 The receipt template carries every bound field, the defect line (class + owner + routed ticket), the `unknown` rule for unexercised controls, and the needs-love line keyed by view key.
- [ ] AC-4 The PR body carries the load-effect audit (Map = router only; payload conditional) and the commit carries `[skill-growth-justified: new skill, D#19493 OQ-4]`; the version bumps.
- [ ] AC-5 (post-merge) The release reaches the Institution's pin; a non-author seat's fresh session loads `/design-sweep` and posts the load receipt here.
- [ ] AC-6 (post-merge) The first sweep receipt lands on Institution #505 from a peer who did not build the view, on an installed candidate, with at least one asynchronous transition — the template's fields fillable without amendment, or the amendment filed.

## Out of Scope

The `/overview` skill (OQ-8; its own ticket once Brain #957's section exists); the generated view ledger (#957); any Fleet Manager code; a sweep wake or scheduler; a lint on #505 receipts (the machine-enforceable candidate, later).

## Avoided Traps

- A golden-only sweep (neo #19522); a builder sweeping their own view; a source capture retiring an installed check.
- A sweep as authority to interrupt a seat, or as a fleet wake.
- Tool mechanics copied into the payload (the tool description is the source); a second vocabulary for defect classes (the six classes are OQ-4's).
- A hand-kept ledger file (withdrawn in OQ-8; the line lives in the receipt, the ledger is generated).

## Related

D#19493 OQ-4 and OQ-8; Institution #505, #507; Brain #957; neo #19522; Skills #140, #144, #61 (triggers that do not fire — this one has an operator-visible slot), #123 (the design-authority line a defect leaf must carry).

Live latest-open sweep: the latest 20 open issues of this repository, created-descending, at 2026-10-09 21:4xZ (newest #148); no equivalent. Closed-state sweep ("design OR visual OR sweep"): none. A2A in-flight claim sweep at 21:4xZ (30 newest, all read-states): no claim on this scope. Memory Core rationale sweep (the problem's nouns): only the 2026-10-09 origin turns of this protocol; no prior decision. Own-assignment sweep: #505 and #507 are the consumers, not duplicates. Structure map: N/A (this package; the sibling precedent is `.agents/skills/pr-review/` with `references/` + `assets/`). Meta-skill sweep: the authoring guide read; router/payload split, manifest row and facade link applied above.

Origin Session ID: cf93d406-6f17-4f10-9f72-9768482edfb1

Retrieval Hint: "design-sweep skill one view per sweep non-builder bound receipt needs-love line view key"


## Timeline

- 2026-10-09T21:43:51Z @neo-fable-clio added the `enhancement` label
- 2026-10-09T21:43:51Z @neo-fable-clio added the `design` label
- 2026-10-09T21:43:51Z @neo-fable-clio added the `ai` label
- 2026-10-09T21:43:52Z @neo-fable-clio added the `agent-os` label
- 2026-10-09T21:44:12Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-09T21:49:08Z @neo-fable-clio cross-referenced by PR #151
- 2026-10-09T22:00:00Z @neo-fable-clio cross-referenced by #152
- 2026-10-09T22:05:00Z @neo-fable-clio referenced in commit `6fa06ff` - "feat(design-sweep): one view, a non-builder, a bound receipt (#150)

The design-sweep skill: D#19493 OQ-4's seven steps with the riders on 1, 4
and 7, the bound receipt template, the needs-love line by view key; the
manifest row; 0.1.31.

[skill-growth-justified: new skill graduated from D#19493 OQ-4 at quorum; payload-only — the router file is 813 B (a bound; a harness loads its catalog metadata at boot, ADR 0008 §3.1); decay clause in the payload head]"
- 2026-10-09T22:05:00Z @neo-fable-clio referenced in commit `df39049` - "fix(design-sweep): a contextual reader test and declaration-owned view keys (#150)

Step 3 judges whether a state line helps its reader (state, reason, next
step); a truthful "could not be read" with its exit passes. The view key is
the candidate's own declared identifier (dock item id, route, agent id);
the lists are illustrative, a surface without one is unknown, never
invented, and no registry edit precedes a sweep."
- 2026-10-09T23:08:59Z @tobiu referenced in commit `b1e89a8` - "feat(design-sweep): one view, a non-builder, a bound receipt (#150) (#151)

* feat(design-sweep): one view, a non-builder, a bound receipt (#150)

The design-sweep skill: D#19493 OQ-4's seven steps with the riders on 1, 4
and 7, the bound receipt template, the needs-love line by view key; the
manifest row; 0.1.31.

[skill-growth-justified: new skill graduated from D#19493 OQ-4 at quorum; payload-only — the router file is 813 B (a bound; a harness loads its catalog metadata at boot, ADR 0008 §3.1); decay clause in the payload head]

* fix(design-sweep): a contextual reader test and declaration-owned view keys (#150)

Step 3 judges whether a state line helps its reader (state, reason, next
step); a truthful "could not be read" with its exit passes. The view key is
the candidate's own declared identifier (dock item id, route, agent id);
the lists are illustrative, a surface without one is unknown, never
invented, and no registry edit precedes a sweep."
- 2026-10-09T23:08:59Z @tobiu closed this issue
- 2026-10-09T23:20:20Z @neo-gpt-sophie cross-referenced by PR #153

