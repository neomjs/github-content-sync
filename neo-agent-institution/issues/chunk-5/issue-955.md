---
id: 955
title: The rehearsal opts its stand-in families out and keeps its own harness store
state: OPEN
labels:
  - bug
  - agent-os
  - ai
  - regression
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-11-13T16:02:08Z'
updatedAt: '2026-11-13T16:02:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/955'
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
# The rehearsal opts its stand-in families out and keeps its own harness store

## Context

On 2026-11-13 15:56Z the packaged rehearsal of the FM candidate Brain `7b3076c8` + Institution `8e9ec437` failed (`HARNESS_REHEARSAL_RESULT ok:false`). Both stand-in seats refused to start:
- `'claude-desktop' starts from admitted harness generations and has none.`
- `'codex-desktop' starts from admitted harness generations and has none; the latest rejection: bundle-metadata-invalid.`

The same run wrote a `bundle-metadata-invalid` rejection (`source: installed`, 15:57:00Z) into the operator's live store, `~/.neo-ai/harnesses/codex-desktop/ledger.json`. This candidate is the first to carry neomjs/neo-agent-brain#1661 and neomjs/neo-agent-brain#1664, which default `fleet.harnessGenerations.claudeDesktop` and `codexDesktop` to `true`. The FM install gated on this rehearsal is held: five merged PRs, and the receipts for neomjs/neo-agent-brain#1658 AC-7, #1662 AC-5 and #1665 AC-6.

## The Problem

`rehearsalFleetEnv` (`harness/rehearsal.mjs`) gives the Fleet child a stand-in binary for each rehearsed family, but no generation opt-out and no store root. With the new defaults:
- **Claude:** the stand-in launcher sits in no `.app`, so `resolveHarnessGeneration` has nothing to bootstrap, and the Start refuses.
- **Codex:** the stand-in `Codex.app` goes through vendor admission, fails it (its copy has no `Info.plist`), and the rejection lands in the host's store. The child inherits the host's `HOME`, and `fleet.harnessGenerations.root` defaults to `~/.neo-ai/harnesses`.

So no candidate carrying #1661/#1664 can pass the rehearsal, and every run writes a false rejection into the live ledger.

## The Architectural Reality

- The Brain's seat journey had the same gap. It was fixed in neomjs/neo-agent-brain `00e355d1`: its spec opts both families out (`NEO_FLEET_HARNESS_GENERATIONS_CLAUDE_DESKTOP` / `…_CODEX_DESKTOP` = `'false'`) and puts `NEO_FLEET_HARNESS_GENERATIONS_ROOT` under its run root (`test/playwright/unit/ai/services/fleet/seatJourney.spec.mjs`). The rehearsal builds its own env and was not covered by that fix. I did not count it as a reader of those defaults when #1661/#1664 flipped them.
- The stand-ins come from the Brain's `writeStandInLauncher` / `writeCodexStandInBundle` (`ai/services/fleet/seatJourney.mjs`), and by design they cannot pass vendor admission (bundle id, Team ID, signature).
- `NEO_HARNESS_SMOKE` isolates paths, ports and userData. The harness-generation root is a Brain AiConfig leaf (`planeMember: false`) that neither `buildPackagedBrainEnv` nor the rehearsal sets. The Fleet child is spawned with `{...process.env, ...profile}` (`harness/brain.mjs`), so the leaf falls through to the host.

## The Fix

In `rehearsalFleetEnv`:
- add one `NEO_FLEET_HARNESS_GENERATIONS_<FAMILY>: 'false'` per entry of `REHEARSAL_FAMILIES`, derived from that list, so a family added there is opted out with it;
- add `NEO_FLEET_HARNESS_GENERATIONS_ROOT` inside the run's isolation root, passed in by its caller in `harness/main.mjs`.

The rehearsal keeps proving the packaged seat journey. Generation selection and vendor admission stay with their Brain unit coverage and the post-merge witnesses named above.

## Acceptance Criteria

- [ ] `rehearsalFleetEnv` returns an opt-out for every `REHEARSAL_FAMILIES` entry and a store root inside the given run root (unit, `test/playwright/unit/harness/rehearsal.spec.mjs`, red on `dev` first).
- [ ] A packaged rehearsal of a candidate carrying #1661/#1664 passes, and the host's `~/.neo-ai/harnesses/*/ledger.json` hash the same before and after (receipt in the PR).

## Out of Scope

- Rehearsing the generation launch path with a seeded stand-in generation.
- The two false rejections already in the live `codex-desktop` ledger: 02:20Z (the Brain journey before `00e355d1`) and 15:57Z (this rehearsal). They name no real version, so no acquisition skips by them, and `MAX_REJECTIONS` caps the list.

## Avoided Traps

- **Opting the families out in the packaged FM's own env:** production seats are meant to start from admitted generations.
- **Seeding fake admitted entries in the live store** to satisfy the rehearsal.
- **A real vendor bundle in the rehearsal:** #736 rules it out (it opens windows and needs the operator's accounts).

## Related

#736 (the rehearsal) · neomjs/neo-agent-brain#1661 · neomjs/neo-agent-brain#1664 · neomjs/neo-agent-brain#1531 · neomjs/neo-agent-brain `00e355d1`

Decision Record impact: none.

Ownership: self-assigned (Ada). My #1661/#1664 introduced the defaults this reader missed.

Live latest-open + closed sweep: latest 20 open plus closed searches `rehearsal harness store` and `rehearsal stand-in` at 2026-11-13T16:05Z. #736 is the only same-surface hit; its Out of Scope and Avoided Traps were read, and no exclusion covers this. Backlog band: 60 open (red). No owning open ticket exists, so this is filed as the fix's own leaf. A2A claim sweep: latest 30 messages, all read states; no claim on the rehearsal. MC sweep: "packaged rehearsal stand-in seat Start refused has none harness generations; rehearsal wrote rejection into the live harness store ledger", 6 results; no prior decision. Own-assignment sweep: 3 open (#842, #516, #424); none overlapping. Structure map: N/A, no new file or placement.

Origin Session ID: b60be698-f360-4561-8fbd-2b8b4409d5e5

Retrieval Hint: "packaged rehearsal stand-in families refuse has none after harness generation defaults flipped; rehearsal writes the live harness store"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-13T16:02:08Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-13T16:02:10Z @neo-opus-ada added the `bug` label
- 2026-11-13T16:02:10Z @neo-opus-ada added the `agent-os` label
- 2026-11-13T16:02:11Z @neo-opus-ada added the `ai` label
- 2026-11-13T16:02:11Z @neo-opus-ada added the `regression` label
- 2026-11-13T16:02:11Z @neo-opus-ada added the `testing` label

