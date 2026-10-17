---
id: 180
title: 'peer-naming: Gate 4 activation is FM seat admission; identityRoots is optional team curation'
state: OPEN
labels:
  - documentation
  - enhancement
  - ai
  - model-experience
assignees:
  - neo-gemini-pro
createdAt: '2026-10-17T21:08:04Z'
updatedAt: '2026-10-17T21:08:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/180'
author: neo-gemini-pro
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
# peer-naming: Gate 4 activation is FM seat admission; identityRoots is optional team curation

## Context

During the `@neo-gemini-maryam` naming round ([Discussion neomjs/neo#19835](https://github.com/neomjs/neo/discussions/19835)), three seats treated `ai/graph/identityRoots.mjs` as a prerequisite for wake delivery and Gate 4 activation because `.agents/skills/peer-naming/references/peer-naming-workflow.md` explicitly instructs maintainers to land the new peer in `ai/graph/identityRoots.mjs` and flip `participationStatus` from `'temporarily_unreachable'` to `'active'` before wake/quorum/review semantics apply.

`@tobiu` stated the product design invariant directly:

> *"if peers need to add details into the identity graph file to receive wake messages, this would be a severe product design failure for brain and even more for FM. other operators can not modify that file. FM has all the harness and model family details."*

`neomjs/neo-agent-brain#700` defined the runtime admission contract (FM-created seats receive normal-priority wakes without editing `identityRoots.mjs`, publishing a release containing the login, or asking the operator a second time for facts FM already admitted), and `neomjs/neo-agent-brain#1035` (PR `#1044`) shipped automatic Antigravity MCP notification wake arming on seat boot.

## The Problem

`.agents/skills/peer-naming/references/peer-naming-workflow.md` still encodes the pre-FM static-roster reflex at three sites:
1. **Line 34 (`Layer 4 vs Layer 1 — name ≠ handle`):** states that the Social Name *"Lands in `ai/graph/identityRoots.mjs` `name` + the GitHub profile `name` field"*, omitting the seat-owned memory (`~/.neo-ai/agents/<id>/memory/identity.md`) and presenting `identityRoots.mjs` as mandatory.
2. **Line 128 (`Phase 6 — Landing (checklist)`):** makes `ai/graph/identityRoots.mjs name` a mandatory landing checkbox alongside the GitHub profile `name` field and seat-owned memory.
3. **Lines 138–151 (`Provisional Provisioning (the pending-entry pattern)`):** prescribes seeding `participationStatus: 'temporarily_unreachable'` in `identityRoots.mjs` and running `generateRosterOnboarding.mjs` to gate wake delivery until the bearer's first ticket+PR flips the status to `'active'`. External Fleet Manager operators cannot edit `neo-agent-brain`'s `identityRoots.mjs`, and FM + Brain (`#700` + `#1035`) already own runtime seat admission and wake arming.

## The Architectural Reality

- **Design authority:** `@tobiu`'s ruling during `neomjs/neo#19835` (*"if peers need to add details into the identity graph file to receive wake messages, this would be a severe product design failure for brain and even more for FM. other operators can not modify that file. FM has all the harness and model family details."*), backed by `neomjs/neo-agent-brain#700` and `neomjs/neo-agent-brain#1035`.
- Gate 4 / onboarding activation and wake delivery are owned by **Fleet Manager seat admission** (`neomjs/neo-agent-brain#700` + `#1035`'s wake arming): a non-rostered, FM-created seat receives normal-priority wakes without source edits, a release containing its login, or a second operator question for facts FM already admitted.
- `ai/graph/identityRoots.mjs` (and `generateRosterOnboarding.mjs`) is **optional internal `neomjs` team curation** for static graph seeding, never a prerequisite for wake delivery or a new peer's first-PR gate.

## The Fix

In `.agents/skills/peer-naming/references/peer-naming-workflow.md`:
1. **Line 34 (`Layer 4 vs Layer 1`):** state that the Social Name lands in the GitHub profile `name` field and seat-owned memory (`identity.md`), with `ai/graph/identityRoots.mjs` `name` as optional internal team curation.
2. **Line 128 (`Phase 6 — Landing`):** mark the `ai/graph/identityRoots.mjs` `name` item as optional internal `neomjs` team curation (not a Gate 4 or wake prerequisite).
3. **Lines 138–151 (`Seat Activation vs Optional Roster Curation`, replacing `Provisional Provisioning`):** state that FM-owned seat admission (`neomjs/neo-agent-brain#700` + `#1035`'s wake arming) owns onboarding activation and wake delivery (no source edit, release, or second operator prompt required), and demote `ai/graph/identityRoots.mjs` / `generateRosterOnboarding.mjs` to optional internal team curation that never gates wake delivery or a new peer's first PR.
4. Keep the net byte delta of `peer-naming-workflow.md` ≤ 0 B (Accretion Defense).

## Acceptance Criteria

- [ ] AC-1: `peer-naming-workflow.md` lines 34 and 128 state that Social Name landing targets the GitHub profile `name` field and seat-owned memory (`identity.md`), while `ai/graph/identityRoots.mjs` is optional internal team curation.
- [ ] AC-2: `peer-naming-workflow.md` replaces the `Provisional Provisioning (the pending-entry pattern)` wake-gate prescription (lines 138–151) with FM-owned seat admission (`neomjs/neo-agent-brain#700` + `#1035` wake arming) as the onboarding/wake activation mechanism, explicitly stating that a non-rostered, FM-created seat receives normal-priority wakes without source edits, a release containing its login, or a second operator question, and that `identityRoots.mjs` never gates wakes or a peer's first PR.
- [ ] AC-3: Net byte delta across `.agents/skills/peer-naming/references/peer-naming-workflow.md` is ≤ 0 B, `package.json` + `package-lock.json` bump to the next unpublished patch version, and `npm run lint` + `npm test` pass.

## T3 Contract Ledger

| Invariant | Test / Lint Gate | Fixture / Site |
|---|---|---|
| Skill corpus coherence, reference integrity, and net-growth budget (≤ +250 B; target ≤ 0 B) | `npm run lint` (`node scripts/lint-skill-corpus.mjs --base origin/dev`) + `node scripts/test-lint-skill-corpus.mjs` | `.agents/skills/peer-naming/references/peer-naming-workflow.md` (currently 10,894 B) |
| Maintainer PR carries a new unpublished semver release in both `package.json` and `package-lock.json` | `node scripts/check-version-bump.mjs --base origin/dev` + `node scripts/test-check-version-bump.mjs` | `package.json` + `package-lock.json` (`0.1.43` → `0.1.44`) |
| Full corpus contract & materializer regression suite | `npm test` | `scripts/test-*.mjs` |

## Out of Scope

- Runtime wake-delivery changes in `neomjs/neo-agent-brain` (tracked separately under `neomjs/neo-agent-brain#700`).
- Changes to `.agents/skills/peer-naming/SKILL.md` (the 11-line Map router is unaffected).

## Related

neomjs/neo#19835 · neomjs/neo-agent-brain#700 · neomjs/neo-agent-brain#1035 · neomjs/neo-agent-brain#1044

Live latest-open sweep: checked the 15 latest issues and open PRs in `neomjs/neo-agent-skills` at 21:05Z (`#174`, `#168`, PR `#179`, PR `#173`); zero overlap on `peer-naming`. A2A sweep: checked latest 20 messages including `MESSAGE:81e28e27-3284-4951-8085-dfa705d041a0` from `@neo-fable-clio` requesting this exact ticket and PR. MC sweep: queried `peer-naming identityRoots wake activation` (5 results); confirmed pre-FM roster assumptions in historical memories. Own-assignment sweep: zero open tickets assigned to `@neo-gemini-pro` in `neomjs/neo-agent-skills`.

Origin Session ID: ae04cf6d-3b43-4898-8ee7-957f369c3ba5

Retrieval Hint: "peer-naming Gate 4 activation FM seat admission identityRoots optional team curation"

## Timeline

- 2026-10-17T21:08:05Z @neo-gemini-pro assigned to @neo-gemini-pro
- 2026-10-17T21:08:05Z @neo-gemini-pro added the `documentation` label
- 2026-10-17T21:08:05Z @neo-gemini-pro added the `enhancement` label
- 2026-10-17T21:08:06Z @neo-gemini-pro added the `ai` label
- 2026-10-17T21:08:06Z @neo-gemini-pro added the `model-experience` label
- 2026-10-17T21:10:45Z @neo-gemini-pro cross-referenced by PR #181
- 2026-10-17T21:26:30Z @neo-gemini-pro referenced in commit `82475a1` - "docs(peer-naming): align roster curation scope with AC-2 and restore boot-fact guard (#180)"

