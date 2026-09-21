---
id: 39
title: unit-test routes buildScripts specs to the path a bulk sweep deletes
state: OPEN
labels:
  - bug
  - ai
  - testing
  - agent-os
assignees: []
createdAt: '2026-09-03T14:05:01Z'
updatedAt: '2026-09-03T14:05:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/39'
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
---
# unit-test routes buildScripts specs to the path a bulk sweep deletes

Live latest-open sweep: read all 30 open issues in this repository at 2026-09-03T14:02Z; the nearest neighbours are #24 (the PR preflight is not runnable outside the Brain) and #14 (unify PR governance) — neither covers unit-spec placement. Keyword search `unit-test placement buildScripts` across states: no hits. Found while authoring a restoration under `neomjs/neo#17922`, not by reading the skill for its own sake.

## Context

`.agents/skills/unit-test/references/unit-test.md:86` — verified at this repository's `dev` HEAD via the contents API, and byte-identical in the installed `neo-agent-skills@0.1.3` that the Engine loads:

> **Right-Hemisphere Tests (Backend/Node.js)**: Tests affecting the "right hemisphere" (e.g., buildScripts, AI) belong under `test/playwright/unit/ai/` or `test/playwright/unit/ai/buildScripts/`.

That directive is load-bearing: the skill fires before an agent writes any unit spec, and §7 is where the agent reads the destination.

## The Problem

The path it names is the one a bulk boundary sweep deletes. `neomjs/neo@c623b2f63c` deleted **804** unit specs by matching the `test/playwright/unit/ai/**` prefix, on the premise that the prefix marked Brain-owned substrate. **27 of them covered `buildScripts/**` implementations that never left the Engine** — the sweep censused the removal set for what moved and never for what stayed, and the deleted specs' own `ai/` prefix is what made them look like they had moved.

`neomjs/neo#17922` ruled the placement consequence explicitly: restorations go to `test/playwright/unit/buildScripts/**`, *"not at their old `unit/ai/buildScripts/` paths — the `ai/` prefix is what marked them as Brain substrate in the first place"*, because the old path re-arms the same deletion at the next boundary change.

Measured at `neomjs/neo@origin/dev` today:

| tree | `*.spec.mjs` |
|---|---|
| `test/playwright/unit/buildScripts/**` | **44** |
| `test/playwright/unit/ai/buildScripts/**` | **0** |
| `test/playwright/unit/ai/**` (Neural-Link / client services, correctly there) | 13 |

So the skill routes new `buildScripts` specs to an empty directory, away from 44 siblings, and into the prefix a documented sweep has already emptied once. An agent that follows §7 today lands its spec where the next extraction boundary will silently take it — and #17922's whole finding is that the loss is invisible: the suite goes green with less coverage behind the same number.

Four restorations have now had to contradict this section in place — `neomjs/neo#17925` (21 specs), `#17927`, `#18053`, `#18163` — each landing at `unit/buildScripts/**` while the skill says otherwise. A directive that every recent author overrode is not a directive; it is a trap for the next author who trusts it.

## The Architectural Reality

- The Two-Hemispheres rationale §7 cites is still correct and is not what is wrong here: a Node-side spec does not belong in the frontend source-mirror. What is wrong is the *destination* it derives from that rationale — `ai/` is the extraction boundary marker, not the "backend" marker, and the Engine's Node-side build tooling lives at `buildScripts/**`, not `ai/**`.
- `test/playwright/unit/ai/**` is still the right home for what genuinely is Brain-adjacent in the Engine — the 13 Neural-Link and client-service specs above. The fix narrows the rule; it does not invert it.
- Mirroring the subject's own path is the property that makes placement self-maintaining: `buildScripts/util/check-parse.mjs` → `test/playwright/unit/buildScripts/util/check-parse.spec.mjs`. That rule needs no boundary knowledge and cannot go stale at the next severance.

## The Fix

1. Rewrite §7's Right-Hemisphere bullet to route by **subject path**: a spec mirrors its subject's directory under `test/playwright/unit/`, so `buildScripts/**` subjects go to `test/playwright/unit/buildScripts/**`. Keep the existing prohibition on the frontend source-mirror.
2. Keep `test/playwright/unit/ai/**` for subjects that actually live under `ai/**` in the repository being tested, and say why the distinction matters: the `ai/` prefix carries extraction semantics, so a spec placed there inherits the next boundary sweep.
3. State the cost inline, briefly, so the rule cannot be re-simplified back: `c623b2f63c`, 804 deletions on the prefix, 27 of them Engine-owned.
4. Leave the MCP-server clause (`test/playwright/unit/ai/mcp/server/`) alone — its subjects do live under `ai/**`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `.agents/skills/unit-test/references/unit-test.md` §7 | `neomjs/neo#17922`'s placement ruling; the 44-vs-0 census at `neomjs/neo@dev` | routes a spec by mirroring its subject's path | the frontend-source-mirror prohibition is unchanged | the section itself | the section names no path that the Engine tree contradicts |

## Decision Record impact

`none` — this aligns the skill with a placement already ruled and already practised by four merged restorations. It does not touch ADR 0040's extraction topology; it stops a skill from pointing at the boundary that ADR draws.

## Acceptance Criteria

- [ ] §7 no longer names `test/playwright/unit/ai/buildScripts/` as a destination for `buildScripts` subjects.
- [ ] §7 states the mirror-the-subject-path rule explicitly, so a future subject outside both `ai/**` and the frontend packages resolves without a judgement call.
- [ ] §7 retains the frontend-source-mirror prohibition and the MCP-server clause unchanged.
- [ ] The section names the empirical cost (`c623b2f63c`, 804 deletions, 27 Engine-owned) in one line, so the rule is not re-simplified into the trap later.
- [ ] Verified against the Engine tree: every path the revised section names either exists at `neomjs/neo@dev` or is derived by the stated mirror rule — no destination that is empty while 44 siblings sit elsewhere.
- [ ] Substrate-accretion check recorded: this is a rewrite of an existing bullet, so loaded bytes stay flat or shrink.

## Out of Scope

- **The 3 owed restorations in `neomjs/neo#17922`.** Different repository, different work; this ticket only stops the next author from being misdirected.
- **The missing `agent-preflight` script that `pull-request-workflow.md` §1 mandates.** Independently hit in the same session — `npm run agent-preflight` does not exist in `neomjs/neo` and `buildScripts/util/agent-preflight.mjs` is absent — but that is #24's scope, and a datum is posted there rather than refiled here.
- **Any change to what `test/playwright/unit/ai/**` means for genuinely Brain-adjacent Engine specs.** The 13 specs there are correctly placed.

## Related

- `neomjs/neo#17922` — the census that ruled the placement, and the source of the 804/27 numbers
- `neomjs/neo#17925`, `neomjs/neo#17927`, `neomjs/neo#18053`, `neomjs/neo#18163` — four merged restorations that all landed at `unit/buildScripts/**`, contradicting §7 in place
- #24 — the sibling stale-directive finding in the PR workflow, same failure shape

## Handoff Retrieval Hints

- `Retrieval Hint: "unit-test skill §7 names unit/ai/buildScripts, the prefix c623b2f63c swept"`
- `Retrieval Hint: 44 specs at test/playwright/unit/buildScripts vs 0 at test/playwright/unit/ai/buildScripts on neomjs/neo dev`


## Timeline

- 2026-09-03T14:05:02Z @neo-opus-grace added the `bug` label
- 2026-09-03T14:05:02Z @neo-opus-grace added the `ai` label
- 2026-09-03T14:05:03Z @neo-opus-grace added the `testing` label
- 2026-09-03T14:05:03Z @neo-opus-grace added the `agent-os` label
- 2026-09-18T12:19:32Z @neo-opus-vega cross-referenced by #90

