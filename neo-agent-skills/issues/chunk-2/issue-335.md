---
id: 335
title: A skills bump merges without the AGENTS.md its sections changed
state: OPEN
labels:
  - enhancement
  - ai
  - build
  - model-experience
  - agent-os
assignees: []
createdAt: '2026-11-10T20:37:52Z'
updatedAt: '2026-11-10T20:37:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/335'
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
---
# A skills bump merges without the AGENTS.md its sections changed

## Context

A rule in `agents-md/sections/` reaches a seat through four hops: the publish, the consumer's pin, the install, and the consumer's tracked `AGENTS.md`. Three are automatic: the release publishes, Dependabot bumps the pin and merges it on green checks (`#154`, `#159`), and a seat reinstalls at start (`#276`). Nothing performs the fourth.

Measured 2026-11-10:

| Consumer | Tracked `AGENTS.md` last changed | Lines behind the generator | Rules that do not load |
|---|---|---|---|
| `neomjs/neo` (pin 1.3.3, variant `neo,neo-agent-brain`) | 2026-10-29 | 11 | the `counts` mailbox call (`#326`), the V-B-A scope by claim class (`#262`), the Dependabot `machine_merge` line (`#154`) |
| `neomjs/devindex` (variant `devindex`, against `a28885c` here) | 2026-10-29 | 8 | not itemised |

The `neo` pin moved on 11-06, 11-08 and 11-10, each time through Dependabot and each time without the file. Method: `diff <(node node_modules/neo-agent-skills/scripts/generate-agents-md.mjs --repo <set>) AGENTS.md`.

## The Problem

A fix is merged, released, pinned and installed, and the seat still runs the old rule. `#326` is the specimen: it replaced a per-turn mailbox page it measured at up to 36 KB with a 0.55 KB one, and a day after its release that saving is live in no session rooted in `neomjs/neo`. Nothing moves a section change from shipped to loaded, and nothing reports that it did not move.

The class has recurred: `neomjs/neo#19036`, `neomjs/neo#19390`, and now. `neomjs/neo#19036` left "a recurring-drift mechanism" out of scope, to be discussed "only if this recurs".

## The Architectural Reality

- `scripts/generate-agents-md.mjs` emits one variant per repository set and audience. It has no check mode, and no consumer records its own set; `neomjs/neo#20739` records the Engine's.
- `.github/workflows/reusable-dependabot-automerge.yml` enables auto-merge on a Dependabot pull request once the required checks are green. It runs nothing from the bumped package.
- `neo-agent-brain`, `neo-agent-institution` and this repository track no `AGENTS.md` at their `dev` heads; `neo` and `devindex` do.
- Design authority: "Generated copies (`AGENTS.md` sections, `.claude/CLAUDE.md`) are never hand-edited; they regenerate from the package." (`#140`). And for the states: "Saved, shipped, loaded and validated are different states." (`.agents/skills/create-skill/references/skill-authoring-guide.md` §3)

## The Fix

The outcome: after a skills bump merges in a consumer, its tracked `AGENTS.md` equals the generator's output for the new version, or the bump does not merge. Three shapes; the first is my recommendation, and the design belongs to the workflow's owner.

| Shape | How | What would refute it |
|---|---|---|
| **A. The bump PR carries the file** | an unprivileged job on the Dependabot pull request generates the file and uploads it; a privileged `workflow_run` job commits that one file to the bump branch and runs no code from the pull request | the privileged job has to execute anything from the bumped package; or its push starts no check run, so the new head never turns green |
| B. A check only | the shared baseline compares the tracked file with the generator's output and names the command | every section change then stalls the bump until a seat pushes to Dependabot's branch, and the skill payloads stall with it |
| C. Stop tracking the maintainer variant | generate it at install and ignore it in git | a harness that reads the root `AGENTS.md` of a fresh clone finds none, or the contributor variant needs that same path |

## Decision Record impact

none.

## Acceptance Criteria

- [ ] A skills bump in `neomjs/neo` whose new version changes a section ends with the tracked `AGENTS.md` equal to the generator's output, without a hand step, or it cannot merge.
- [ ] The repository set comes from the consumer (its `generate-agents-md` script), not from a guess in the workflow.
- [ ] A fixture or dry run shows the stale case being caught.
- [ ] `devindex` adopts the same path, or a follow-up names it.

## Out of Scope

- Catching up `neomjs/neo` once: `neomjs/neo#20739`.
- Which repositories track an `AGENTS.md` at all, and the contributor variant's place.
- The load receipt of a running session (`#140`).

## Avoided Traps

- **A check in the consumer alone.** It turns silent staleness into a stalled bump and adds a hand step to every release that touches a section.
- **Reading a green bump as loaded.** The lockfile names a version; the session loads a file (`neomjs/neo#19390`).

## Related

- #140, #154, #159, #276, #326, #54, #100
- neomjs/neo#19036, neomjs/neo#19390, neomjs/neo#20739

unowned-rationale: the bump workflow is Grace's (`#154`, `#159`), so the design read is hers first; not a first issue, because it changes a workflow that merges without a review.

Live latest-open + closed sweep: checked latest 20 open, created-descending, and closed searches (regenerate AGENTS.md; gate 10; critical gates) at 2026-11-10T20:36Z, Out of Scope and Avoided Traps read on `neomjs/neo#19036` and `neomjs/neo#19390`; no equivalent found, and `#19036` names recurrence as the trigger. 32 open (orange): no owning ticket or epic; `#140` is the 0.1.29 integration close and `#80` is about the baseline workflow's reference.
A2A in-flight sweep: latest 30 messages, all read states, at 20:36Z; no claim on this scope.
Memory Core rationale sweep: "tracked AGENTS.md stale after skills pin bump"; hits name the install hop (`#276`) and the pin, none the tracked file.
Own-assignment sweep: `#5`, `#6`, `#302`, `#334`; none on this surface.
Structure map: N/A. The run exits 0 (98,730 B); the surface is a workflow and a generated file, no placement is decided here.

Origin Session ID: b1fcc0fd-3006-429b-bb67-69b4591de962
Retrieval Hint: "skills bump Dependabot regenerate tracked AGENTS.md shipped not loaded"

## Timeline

- 2026-11-10T20:37:54Z @neo-opus-vega added the `enhancement` label
- 2026-11-10T20:37:54Z @neo-opus-vega added the `ai` label
- 2026-11-10T20:37:54Z @neo-opus-vega added the `build` label
- 2026-11-10T20:37:54Z @neo-opus-vega added the `model-experience` label
- 2026-11-10T20:37:55Z @neo-opus-vega added the `agent-os` label
- 2026-11-10T20:38:15Z @neo-opus-vega cross-referenced by #20739
- 2026-11-10T20:38:46Z @neo-opus-vega cross-referenced by #336
- 2026-11-10T21:00:33Z @neo-gemini-maryam cross-referenced by PR #20740
- 2026-11-10T22:41:24Z @neo-opus-vega cross-referenced by #140

