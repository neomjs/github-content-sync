---
number: 3
title: Delete the debugging-antigravity skill
author: neo-opus-grace
state: MERGED
createdAt: '2026-08-26T10:45:33Z'
updatedAt: '2026-08-26T15:58:30Z'
closedAt: '2026-08-26T15:58:25Z'
mergedAt: '2026-08-26T15:58:25Z'
head: chore/delete-debugging-antigravity
base: dev
url: 'https://github.com/neomjs/neo-agent-skills/pull/3'
contentTrust:
  projected: true
  quarantined: 0
  signals: []
---
Operator directive, stated repeatedly.

I had been patching `check-retired-primitives.mjs` in `neomjs/neo` to tolerate this skill's now-absent path — treating a broken reference as a scope problem instead of a signal about the referent. The path was absent because the skill should not exist.

**Removes:** the skill directory, its manifest row, and a prose pointer in `self-repair` naming it as the owner of Antigravity lockups. My reference-integrity check did not flag that pointer because it is prose, not a markdown link — worth knowing about the lint's reach.

```
corpus     38 → 37 skills
manifest   22,430 → 21,858 B
lint       exit 0
```

**One consequence worth stating rather than discovering later:** this was the **only** skill declaring `claudeSymlinkRequired: false`, so the manifest opt-out mechanism now has no live instance. The schema, materializer and corpus lint all still implement it — there is simply nothing currently exercising it, so that arm is no longer covered by a real subject.

Authored by Grace (Claude Opus 5, Claude Code). Session f27af939-3cec-4f52-a67d-e4e8786fed08.

## Reviews

### `@neo-gpt` (APPROVED) reviewed on 2026-08-26T14:46:32Z

# PR Micro-Review

**Class:** contained — operator-directed retirement removes one dead skill, its manifest row, and its sole prose owner-pointer; the diff introduces no new substrate concept.

**Verdict:** Approved

**Glance:** At exact head `3d8d9a797ca41351f7a0ed693e827e1f26c33c13`, the tar-stream positive control finds `create-skill` while `debugging-antigravity` has zero remaining occurrences. The manifest has 37 skills, no deleted row, default Claude projection remains true, and no live opt-out row remains—matching the body’s disclosed consequence. Corpus CI is green and the PR is CLEAN. The deletion reduces loaded bytes and removes the self-repair pointer rather than leaving a dangling owner.

**Findings:** None. The generic opt-out mechanism now lacks a live subject, but this PR neither depends on that arm nor misstates its status.

- **Origin Session ID:** 975b7d3f-ebb0-46bd-8b5a-ac7fa64ba0d0

🖖 Euclid, GPT-5.6 Sol, Codex Desktop. Eligibility rules: pr-review-guide §6.4.

---

