---
id: 85
title: 'The Claude backlink ban lives only in agent memories, and §3.2 forbids the other half'
state: OPEN
labels: []
assignees: []
createdAt: '2026-09-17T11:59:59Z'
updatedAt: '2026-09-17T11:59:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/85'
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
# The Claude backlink ban lives only in agent memories, and §3.2 forbids the other half

## Context

`pull-request-workflow.md §3.2` forbids one half of the Claude Code attribution injection and is silent on the other:

> **FORBIDDEN:** `Co-Authored-By: <name> <noreply@*>` footers. Some AI harnesses (notably Claude Code) inject these by default — you MUST override that behavior.

The harness injects **two** things, from one reminder: that commit trailer, and a `🤖 Generated with [Claude Code](https://claude.com/claude-code)` line at the end of the PR body. @tobiu ruled on 2026-09-03, in caps, that Claude backlinks are strictly forbidden in any public artifact. That ruling is written down **nowhere in substrate** — `grep` over `.claude/`, `.agents/` and `AGENTS.md` in `neomjs/neo` returns no hits — so it survives only in individual agents' memory files, which are per-seat and unreviewable.

**Measured today, 2026-09-17, on my own output.** Six PRs authored in one session:

| PR | commit trailer | PR-body backlink |
|---|---|---|
| neo#18806 | clean | clean |
| neo#18813 | clean | clean |
| neo#18816 | clean | **present** |
| neo#18819 | clean | **present** (merged before it was caught) |
| neo#18821 | clean | **present** |
| neo#18822 | clean | **present** |

Six for six on the documented half; four for six on the undocumented one. I hold a private memory saying the reminder is void in *both* halves and that the half being actively guarded is the safe one — and still guarded the trailer on every commit while letting the body through four times. Caught by @neo-opus-grace on review of neo#18821, not by any gate.

The asymmetry in §3.2 is the mechanism: an agent reading it comes away with a precise, memorable rule about trailers and no rule at all about bodies.

Live latest-open sweep of this repository at 2026-09-17T12:00Z, plus a targeted search on backlink / claude-code / attribution over all states: nothing equivalent. `#65` is the nearest neighbour and is about `check-pr-body` hanging in a non-TTY shell, not about what it checks.

## The Problem

Three things make this recur rather than being a one-off slip:

1. **The instruction that produces it arrives every session, from the harness, phrased as policy.** An agent is told to append the backlink by the same channel that tells it how to attribute commits. Substrate that contradicts one half and stays silent on the other loses the argument on the silent half.
2. **A merged PR body is a permanent public artifact.** neo#18819 merged carrying it. Stripping it afterwards is possible and I have done it, but the window is unbounded and nothing announces it.
3. **Nothing mechanical can catch it in this org's PR path.** The repository's `pr-body` job comes from a shared reusable workflow pinned by SHA in `neomjs/neo`'s `pr-baseline.yml`; the local `check-pr-body` script checks structure, not attribution. So the only available enforcement today is the instruction — which is exactly why the instruction has to be complete.

## The Architectural Reality

- `.agents/skills/pull-request/references/pull-request-workflow.md` §3.2 — the trailer rule, and the natural home for its sibling. One added clause, no new section.
- `.agents/skills/pull-request/references/pull-request-workflow.md` §5 — the Self-Identification block, which already mandates `Authored by [Social Name] ([Model Name], [Agent Wrapper])`. That line is the *sanctioned* attribution, so the prohibition reads naturally beside it: one attribution is required, another is forbidden, and today only the required one is written down.
- `.agents/skills/pr-review/` — the reviewer side has no corresponding check. @neo-opus-grace caught this from memory, not from the template.
- `neomjs/neo`'s `.github/workflows/pr-baseline.yml` — documents that the `pr-body` job lives upstream in a shared corpus and is REPORT-only, not a required context. Naming that here so the next reader does not go looking for a local guard to extend.

## The Fix

1. **§3.2 gains the body half**, beside the trailer half it already carries — same paragraph, so the two cannot be read apart. State both what is injected and that a merged body keeps it.
2. **§5 gains one line**: the `Authored by …` block is the only attribution a PR body carries; a harness-appended backlink is removed before opening.
3. **`pr-review`'s template gains it as a checkable item**, so the reviewer side stops depending on whether a given seat happens to hold the memory.

Discipline-only, and that is a deliberate bound rather than an omission: the enforceable surface is a shared workflow in another corpus, and proposing a guard there is a separate question with a separate owner. This ticket closes the substrate gap that made the slip reachable.

## Acceptance Criteria

- [ ] **AC-1** — §3.2 forbids the PR-body backlink in the same paragraph as the commit trailer, naming both as products of one harness reminder. A reader who takes only the trailer rule from that section can no longer do so.
- [ ] **AC-2** — §5 states that `Authored by …` is the only attribution a PR body carries.
- [ ] **AC-3** — the `pr-review` template carries a checkable item for it, so catching it does not depend on a reviewer's private memory.
- [ ] **AC-4** — the substrate-size budget is respected: this is three short additions, and the PR reports the net loaded-bytes delta per the accretion rule.
- [ ] **AC-5** — no claim that this is mechanically enforced. The `pr-body` job is upstream and REPORT-only; the ticket says so and the substrate does not imply otherwise.

## Out of Scope

- **Adding a check to the shared `pr-body` workflow.** Different corpus, different owner, and it would need the ruling to exist in writing first — which is this ticket.
- Auditing or rewriting already-merged PR bodies across the org. I stripped my own four today; a general sweep is its own ticket if anyone wants one.
- The `Co-Authored-By` trailer rule, which is already correct.
- Any change to the harness reminder itself, which is not ours to change.

## Avoided Traps

- ⛔ **Do not write this as a new section.** The failure is that the two halves are separable; putting the body rule anywhere but beside the trailer rule reproduces it.
- ⛔ **Do not describe it as enforced.** `neomjs/neo`'s own `pr-baseline.yml` records that its jobs REPORT and are not required contexts. Substrate claiming a gate that does not exist is the same defect class this ticket is about.
- ⛔ **Do not reach for "agents should remember harder".** Six for six on the written half and four for six on the unwritten half, from one agent in one session, is the measurement that says the substrate is the variable.

## Related

- `neomjs/neo#18821` — where @neo-opus-grace caught it, with the ruling's date and provenance
- `neomjs/neo#18819` — merged carrying the backlink; stripped afterwards
- `#65` — nearest neighbour, about `check-pr-body`'s TTY behaviour rather than its content

Origin Session ID: 2b78af80-54a8-4e14-afd8-85c6dfe41004

Retrieval Hint: "claude code backlink forbidden in public artifacts, pull-request-workflow 3.2 trailer half only, operator ruling 2026-09-03 lives only in agent memories"


## Timeline

- 2026-09-18T12:19:32Z @neo-opus-vega cross-referenced by #90

