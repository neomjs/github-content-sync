---
id: 366
title: Home's field heals after a click and holds its density
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-vega
createdAt: '2026-09-30T18:26:04Z'
updatedAt: '2026-09-30T21:43:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/366'
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
closedAt: '2026-09-30T21:43:27Z'
---
# Home's field heals after a click and holds its density

## Context

Grace's motion observation of #360 at `edd0bc1` ([issuecomment-5916457647](https://github.com/neomjs/neo-agent-institution/pull/360#issuecomment-5916457647), 2026-09-30) measured two defects in Home's field. The probe read the renderer's `getStats()` through the App worker and counted bright motes inside fixed circles on timed screenshots. Neither defect blocks #360, whose AC-1..AC-4 hold. The figures below are Grace's, and the code reading is mine, at `edd0bc1`. The third finding, the loop running while the window is hidden, belongs to the engine's `SharedCanvas` and is filed there as neomjs/neo#19336.

## The Problem

1. **A click leaves a hole.** Bright motes within 250 px of a click went 18 → 7 (+2 s) → 3 (+10 s) → 8 (+40 s), while a control circle held 15–19. The pointer's parting leaves the same trail behind it.
2. **The field thins over minutes.** Untouched at 1392×850, the 20-circle total went 69 → 34 over 180 s. The goldens capture the seeded frame, which is the densest one the field ever shows.

## The Architectural Reality

- `AgentOS.canvas.Home#step` (`apps/agentos/canvas/Home.mjs:653`): the pointer (`:695`) and each ripple (`:706`) add velocity to the motes they reach. The current's `pull` relaxes velocity back toward the flow's 5–15 px/s, and nothing restores position, so a flung mote returns only by drift.
- `#seed` (`:565`) places motes in `[0, width] × [0, height]`. `#step` wraps them over `[-link, width + link] × [-link, height + link]` (`:728`), with a margin that keeps a wrapping mote's links off screen. At 1392×850 the visible share of that domain is 1392·850 / (1632·1090) = 66.5 %, so the steady state keeps about two thirds of the seeded density on screen. That predicts −33.5 %, and Grace measured −51 %. The wrap domain is one cause; the rest is not yet explained.

## The Fix

1. Ripples and the pointer's parting displace the **drawn** position with a decaying offset and leave the motes' state untouched (Grace's suggestion). Once a ripple's `life` has passed, the field under the click is the field without it, and the same holds for a region the pointer leaves.
2. The field holds its design density on screen: seed across the wrap domain, with the count and `moteMax` scaled to that domain's area, or whatever change the measurement shows also closes the unexplained remainder. `drawLinks` is O(n²) over the motes, so 1.5× the motes costs about 2.25× the pair checks, and the arm records frame time beside density.

## Acceptance Criteria

- [ ] AC-1 **No lasting hole.** Once a ripple's `life` has passed, the motes drawn within 250 px of the click number at least 85 % of the same region in an unclicked control stepped identically (today at +10 s: 3 against 15–19). The same holds 1 s after the pointer leaves a region it parted. Renderer unit arm, fixed `dt`.
- [ ] AC-2 **The density holds.** Untouched at 1392×850, the visible mote count stays within 10 % of the design density, `width · height / density` (197), in the seeded frame and after 180 s stepped, and within 15 % at every second between. The instantaneous count dips as the current carries motes, so no stricter continuous bound is claimed. `getStats` reports the visible count, and the arm records the frame time. If the seeded frame changes, the goldens are re-recorded and reviewed by the design owner.
- [ ] AC-3 **Every size stays safe.** A resize to zero width or zero height seeds no motes and throws nothing, and a sliver such as 300×1 stays within `moteCap`, which bounds the links' pairwise work.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `field.part` (a share; replaces the velocity `push`) | AC-1; Grace's design review on #368 | The pointer clears a core of `part × reach` and maps the disc onto the ring outside it by area: a flat 1.33× density at 0.5 | The parting eases out after the pointer leaves (under 6 % after 1 s) | JSDoc | `home.spec.mjs`, AC-1 arm |
| `field.ripple.lift` (px; replaces the velocity `ripple.push`) | AC-1 | A ripple's band lifts drawn positions by up to `lift`, decaying over `life` | Nothing after `life` | JSDoc | AC-1 arm: state equals the untouched control |
| `field.moteMin` / `moteMax`: visible-count targets, not per-frame bounds | AC-2 | The visible target clamps to `[moteMin, moteMax]`, and the total scales it to the wrap domain | See `moteCap` | JSDoc | AC-2 arm |
| `field.moteCap` (new): a hard cap on the total | AC-3 | At most 320 motes on any surface, which bounds the links' pairwise work | A zero-area surface holds none | JSDoc | `home.spec.mjs`, resize arm |
| `getStats().visible` (new) | AC-2 | The motes the last frame drew inside the surface | `0` before any frame | JSDoc | AC-2 arm |

## Out of Scope

- The quiet rects under the hero's lines (#365). Its Out of Scope line on thinning near the lines is about a different thinning.
- The loop while the window is hidden: the engine's `SharedCanvas` owns renderer lifecycle (neomjs/neo#19336), and #367 carries it here and checks it on the installed build.
- Retuning the field's look (speeds, radii, link alpha) beyond holding its density.

## Avoided Traps

- **A spring back to a rest position:** it gives every mote a home, and the field stops drifting as a current.

## Related

#244 · #360 · #365 · neomjs/neo#19336

Decision Record impact: none.
Structure map: N/A. The change stays in one existing Institution app file, which the Brain's structure map does not host.
Live latest-open sweep: the latest 20 open issues at 2026-09-30T18:25:21Z; no equivalent found.
A2A in-flight claim sweep (all read states, last 30): no claim on Home's field; Grace offered to file it and left it to me.
MC sweep: "Home field click leaves a hole motes thin over minutes hidden window keeps the loop running canvas worker", 6 results, no prior decision found.
Own-assignment sweep: 2 open (#365, #244); #365 is the sibling named in Out of Scope, and #244 holds only its pointer to #365.

Starts after #360 lands, since it changes the renderer #360 adds.

Origin Session ID: eee527e0-79de-4edb-b4c2-eb89d3780d56
Retrieval Hint: `query_raw_memories("Home field click hole ripple thinning wrap domain density motes")`



## Timeline

- 2026-09-30T18:26:05Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-30T18:26:06Z @neo-opus-vega added the `bug` label
- 2026-09-30T18:26:07Z @neo-opus-vega added the `agent-os` label
- 2026-09-30T18:26:07Z @neo-opus-vega added the `ai` label
- 2026-09-30T18:26:07Z @neo-opus-vega added the `design` label
- 2026-09-30T19:22:23Z @neo-opus-vega cross-referenced by PR #19338
- 2026-09-30T19:45:43Z @neo-opus-vega cross-referenced by #367
- 2026-09-30T19:47:10Z @neo-opus-vega cross-referenced by PR #368
- 2026-09-30T19:52:40Z @neo-opus-vega referenced in commit `f09c777` - "chore(agentos): merge dev into Home's field heals, re-stamped on the merged tree (#366)"
- 2026-09-30T20:03:36Z @neo-opus-vega referenced in commit `fd1ad37` - "fix(agentos): the pointer parts Home's field by area, so no wall piles at its rim (#366)"
- 2026-09-30T20:07:18Z @neo-opus-vega referenced in commit `957c09c` - "test(agentos): re-stamp the visual baselines over the area-preserving parting (#366)"
- 2026-09-30T21:11:29Z @neo-opus-vega referenced in commit `b323444` - "fix(agentos): Home's field stays bounded on every size a resize can deliver (#366)"
- 2026-09-30T21:11:29Z @neo-opus-vega referenced in commit `a58807c` - "test(agentos): re-stamp the visual baselines over the bounded seed (#366)"
- 2026-09-30T21:18:30Z @neo-opus-vega referenced in commit `bf8015e` - "fix(agentos): Home's field documents its density as a target the count drifts around (#366)

"Does not thin it" still read as a per-frame guarantee. At 1392×850 with a fixed
dt, the visible count is 193 seeded and 182 at 180 s, and it swings from 175 (129 s)
to 208 (47 s). The JSDoc now states the flux balance that holds the target and says
the count drifts around it. Re-stamped after the Home goldens passed 2/2."
- 2026-09-30T21:43:27Z @tobiu referenced in commit `d058a63` - "Merge pull request #368 from neomjs/vega/366-home-field-heals

fix(agentos): Home's field heals after a click and holds its density (#366)"
- 2026-09-30T21:43:27Z @tobiu closed this issue
- 2026-09-30T22:12:54Z @neo-gpt-emmy cross-referenced by PR #372
- 2026-09-30T22:15:39Z @neo-gpt cross-referenced by PR #373

