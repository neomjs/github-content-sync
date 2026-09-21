---
id: 90
title: 'The published guards have no commit-time installer, so only neo runs them — as divergent local copies'
state: OPEN
labels:
  - enhancement
  - contributor-experience
  - ai
  - build
assignees: []
createdAt: '2026-09-18T12:19:31Z'
updatedAt: '2026-09-18T12:20:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/90'
author: neo-opus-vega
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
# The published guards have no commit-time installer, so only neo runs them — as divergent local copies

## Context

@tobiu, 2026-09-18, on neo-agent-brain#370 (merged): *"it might need a follow-up ticket inside the skills repo => in case we want to make the hooks work inside multiple org repositories."*

#370 deleted the Brain's orphaned hook-CI parity guard because the Brain has no hooks for it to read. That is the symptom; this is the question behind it. Measuring before proposing changed what the ticket should say, so the measurement comes first.

**The hook plane across the six org repos, read from each checkout at 2026-09-18:**

| repo | `.husky/` | `lint-staged` keys | calls `reusable-pr-baseline` |
|---|---|---|---|
| `neo` | `pre-commit`, `pre-push` | 9 | yes |
| `neo-agent-brain` | — | 0 | **no** (#40) |
| `neo-agent-skills` | — | 0 | n/a — it is the publisher |
| `neo-agent-institution` | — | 0 | yes |
| `devindex` | — | 0 | yes |
| `create-app` | — | 0 | no |

**And what `neo`'s hooks actually run is the part that reframes this.** Its `lint-staged` block invokes `buildScripts/util/check-*.mjs` — **engine-local scripts, not one published bin**. This package ships seven bins (`…-pr-body`, `…-substrate-size`, `…-ticket-archaeology`, `…-workflow-concurrency`, `…-npm-overrides`, `…-materialize`, `…-agents-md`), and the only thing that runs any of them is `reusable-pr-baseline.yml` in CI.

So the accurate statement is not *"hooks do not work in multiple repos."* It is:

- **No repo runs a published guard at commit time.** `neo` included.
- `neo` has commit-time guards because it hand-rolled a parallel set, and those copies have already drifted from the published ones — filed as neomjs/neo#18809, *"The engine's archaeology mirror and the published guard disagree, and the hook is the permissive side."* A second implementation is the cost we are already paying.
- The other five repos have no commit-time gate at all, so `git commit` there is unguarded and `--no-verify` is not even the interesting case.

## The Problem

A guard that runs only in CI is a guard that reports after the work is pushed. That is fine for arbitration and poor for feedback, and it is why `neo` built local copies rather than wait. The package solved distribution for the CI half (a pinned `npm install` from `runner.temp`, per-job, immutable) and did not solve it for the commit half, so each consumer either reimplements or goes without.

The divergence is the compounding cost. neomjs/neo#18809 shows the two archaeology implementations disagreeing **with the hook on the permissive side** — the copy that runs earliest is the weaker one, which inverts the point of running early.

## The Architectural Reality — and the boundary this ticket must not cross

**Agent OS hooks are governed and out of scope.** ADR 0040 §2.7 rules that lane-state, wake, presence and Memory-Core context hooks are Agent OS substrate, that **seat provisioning materializes them into target checkouts as generated-not-tracked artifacts**, and §2.5 that they resolve Brain substrate only through `agentosRuntimeRoot`, never relatively from `targetRepoRoot`. That is leaf 11, implemented under neo-agent-brain#250.

**This package already tried to own that half and was told no.** #21 — *"Hooks ride the skills materializer instead of a config-string merge"* — is **CLOSED / NOT_PLANNED**, superseded by ADR 0040 §2.5/§2.7 and brain#250. Anyone re-reading @tobiu's steer in #21's body (*"non repo specific ci workflows and hook should work the same way… and installers"*) should read it alongside that closure: the Agent-OS-hook half was routed to seat provisioning, deliberately.

ADR 0040 §2.7 also draws the line this ticket lives on: *"Engine-only contributor guards that carry no Brain dependency stay Engine-owned"* — and test and hook custody **follows the subject, not the directory**. The seven bins here are contributor guards with no Brain dependency. They are the other family, and nothing governs their commit-time distribution.

- **Existing distribution precedent to follow:** `.github/workflows/reusable-pr-baseline.yml` installs each guard from `runner.temp` at a pinned `SKILLS_VERSION` and invokes it bare in the caller workspace, so a PR cannot relax the guard judging it. A commit-time installer has the same property to preserve and a harder constraint: it runs from the consumer's own `node_modules`.
- **`neo`'s `prepare` script** (`node ./buildScripts/util/prepare.mjs`) is the existing seam where husky is wired; the other five repos have no `prepare` at all.
- **Out-of-date risk is the reason a pin matters:** a hook pinned to a floating version changes a contributor's local gate without a PR.

## The Fix

Not prescribed here — the shape is a real fork and this ticket exists to hold the measurement while it is decided. The candidates, with what would settle each:

1. **A `neo-agent-skills-install-hooks` bin** that writes `.husky/` entries invoking the published bins, run from a consumer's `prepare`. Settles if consumers want husky; costs a generated-vs-tracked decision per repo.
2. **A published `lint-staged` fragment** consumers spread into their own block. Smaller, but does nothing for a repo with no husky at all — which is five of six.
3. **Do nothing for five repos and converge `neo` instead** — retire the engine-local copies onto the published bins, closing neomjs/neo#18809 by construction rather than by fixing the divergence. Cheapest, and it fixes the case that is actively wrong today.

Option 3 deserves naming first because it is the only one that removes code.

## Acceptance Criteria

- [ ] The chosen option is recorded here with the falsifier that selected it, before any implementation.
- [ ] Whatever ships, `neo`'s archaeology hook and this package's `…-ticket-archaeology` bin no longer disagree — verified by running both against the same staged fixture and comparing verdicts, not by reading them.
- [ ] A consumer's commit-time gate names a pinned version; a floating pin is a Required Action at review.
- [ ] The ADR 0040 §2.7 boundary is restated in whatever lands, so the Agent-OS-hook half is not re-litigated by the next reader.

## Out of Scope

- **Agent OS hooks** — lane-state, wake, presence, Memory-Core context. ADR 0040 §2.7 / leaf 11 / brain#250 own them, and #21 is the record of this package not owning them.
- **The CI half for `neo-agent-brain`** — #40 already holds *"neo-agent-brain calls no PR baseline, so five shipped guards never run."* This ticket is the commit-time half and must not absorb it.
- **Adding guards.** Nothing here proposes a new check; it is about where the existing ones run.

## Avoided Traps

- **Reading @tobiu's 2026-09-18 remark as a fresh requirement.** It restates a steer recorded in #21's body on 2026-08-30, whose Agent-OS half was then decided against this package. Filing without that closure would have re-opened a settled question.
- **"Make the hooks work in multiple repos" as the frame.** It presumes `neo`'s hooks are the thing to spread; they run local copies that are already the permissive side of a known divergence, so spreading them would spread the defect.
- **Treating the absence of `.husky/` in five repos as the whole gap.** `neo` has hooks and still runs zero published guards, so the gap exists at six of six.

---

**Live latest-open sweep:** latest 20 open issues in this repo read at 2026-09-18T12:16Z — #87, #86, #85, #80, #76, #66, #65, #64, #63, #62, #61, #60, #59, #56, #51, #40, #39, #38, #37, #23. Nearest are #40 (CI half, scoped out above) and #51 (agent shell guard, a different subject). An all-states search for `husky OR hook OR lint-staged` surfaced **#21 CLOSED/NOT_PLANNED**, which is the governing prior art and is cited above rather than duplicated.
**A2A in-flight sweep:** `list_messages({status:'all', limit:8})` at the same time — recent claims are neo#18852, #18857, #18869, Institution #152 and brain#370. None touches hook or installer scope.
**Own-assignment sweep:** none in this repo.

Retrieval Hint: "published guards run only in CI; neo runs divergent local copies at commit time; ADR 0040 §2.7 owns the Agent OS hook half"

Origin Session ID: 2baab8f1-a2b2-4183-b903-8d7231796cab


## Timeline

### @neo-opus-vega - 2026-09-18T12:20:23Z

## Splitting the fork by tier — option 3 is not blocked on anyone

I filed this leaving the whole fork open for a decision above my tier. That was wrong on inspection: only part of it is.

- **Options 1 and 2 install a commit-time gate into repositories that have none.** That is a cross-cutting mutation, which `AGENTS.md §swarm_topology_anchor` excludes from Tier 2 — it routes to Tier 3 (Ideation Sandbox), not to an operator ask. It stays open here, unclaimed, until someone takes it there.
- **Option 3 — converge `neo` onto the published bins — is Tier 2.** One repository, reversible in one commit, no API break, no cross-cutting mutation, and it *removes* a second implementation rather than adding a plane. It also closes neomjs/neo#18809 by construction instead of fixing the divergence it reports.

So option 3 is claimable now and does not wait on the rest. I am not claiming it in this comment — I have two review seats in front of it — but nothing else should treat it as blocked, and whoever reaches it first should take it.

The measurement above stands for all three options either way.

— Vega (Claude Opus 5, Claude Code) 🌿


- 2026-09-18T16:28:35Z @neo-opus-ada cross-referenced by #91
- 2026-09-19T15:40:21Z @neo-opus-ada cross-referenced by #93

