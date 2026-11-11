---
id: 349
title: Skill-authoring guide names a missing lint and a retired symlink step
state: OPEN
labels:
  - bug
  - documentation
  - good first issue
  - ai
  - model-experience
assignees: []
createdAt: '2026-11-11T19:03:44Z'
updatedAt: '2026-11-11T19:03:44Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/349'
author: neo-fable-mnemosyne
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
# Skill-authoring guide names a missing lint and a retired symlink step

## Context

Found on 2026-11-11 while reading the `create-skill` payload in full. Every seat reads this guide before it creates or changes a skill, so a wrong step in it is followed literally.

## The Problem

Read at `dev` `b3a73a9` (1.4.0), `.agents/skills/create-skill/references/skill-authoring-guide.md`.

**1. The verification list ends with a command that cannot run.** Line 180 asks:

```
$ node ai/scripts/lint/lint-skill-manifest.mjs --base origin/dev
Error: Cannot find module '<repo>/ai/scripts/lint/lint-skill-manifest.mjs'
```

The file exists in no repository: checked in clones of `neomjs/neo-agent-skills`, `neomjs/neo` and `neomjs/neo-agent-brain` at their `dev` heads on 2026-11-11. The lint this repository runs is `scripts/lint-skill-corpus.mjs`, wired as `npm run lint`, and it passes today ("40 skills, coherent with the manifest, within budget"). The path was corrected once before, in neomjs/neo#12917 (June), and went stale again when the corpus moved into this repository.

Line 136 names the same missing script as the home of the budgets. `perFilePayloadBudget` and `maxPositiveDeltaBytes` live in `scripts/lint-skill-corpus.mjs` (lines 546 and 669). Line 139 names a third check, `checkSectionTriggers`; that name appears in no script of this repository.

**2. Section 4 mandates a link by hand that the package makes.** Lines 158 to 169 and the checklist item at line 179 tell the author to run `ln -sf ../../.agents/skills/my-new-skill .claude/skills/my-new-skill`, as "CRITICAL". This repository has no `.claude/` directory. In a consumer, the materializer projects `.claude/skills` as per-skill links from the manifest, untracked. An author who follows section 4 in a consumer creates a tracked link there, which the README calls a shadow copy.

## The Architectural Reality

- The guide is the payload of the `create-skill` router. It loads only when a seat creates or modifies a skill.
- `scripts/lint-skill-corpus.mjs` is this repository's corpus lint (`npm run lint`), covered by `scripts/test-lint-skill-corpus.mjs`.
- `scripts/materialize-harness-skills.mjs` owns both projections. The manifest's `claudeSymlinkRequired: false` is the only per-skill opt-out (`README.md:329-331`).
- Design authority: "`postinstall` materializes **two surfaces**, both as symlinks into `node_modules`, both untracked" and "a tracked entry under either is a shadow copy, which is the thing this package exists to make unnecessary" (`README.md:21`, `:30-31`).

## The Fix

In `skill-authoring-guide.md` only:

- line 180: name `npm run lint`, run in this repository;
- line 136: name `scripts/lint-skill-corpus.mjs`;
- line 139: name the check as the lint names it, or drop the bullet if the lint has no such check;
- section 4 and checklist line 179: say what is true. The author adds the manifest row; the materializer projects the Claude façade in every consumer; `neo-agent-skills-materialize --check` verifies it. The section gets shorter.

The router is untouched and the payload shrinks, so the loaded bytes do not grow.

Decision Record impact: none. Contract Ledger: not applicable. No surface changes; four instructions are corrected to match what ships.

## Acceptance Criteria

- [ ] Every command the guide names runs in a fresh clone of this repository after `npm ci`.
- [ ] The guide contains no instruction to create a link under `.claude/skills/` by hand.
- [ ] The guide names the manifest row and `neo-agent-skills-materialize --check` as the way a new skill reaches the Claude façade.
- [ ] Every check the guide names by name exists under that name in `scripts/lint-skill-corpus.mjs`.
- [ ] `npm run lint` and `npm test` pass, and the payload's byte count does not grow.

## Out of Scope

The router baseline of 7 to 12 lines "across all 18 current skills" (line 71): it is a dated measurement, there are 40 skills today, and it wants a new measurement, not an edit. The wording of the router template. Any change to the lint itself.

## Avoided Traps

- **Restoring a script at the old path.** The lint has a home and a name; the guide follows it.
- **Keeping section 4 "for safety".** A link made by hand and tracked is the shadow copy the package exists to remove.

## Related

#24 (closed): a mandated command that could not run outside one repository. Same class, different skill.

unowned-rationale: Ready for a contributor; maintainers provide planning and review. Four lines and one section in one payload file.

Live latest-open + closed sweep: latest 20 open of 33 (orange band; no open ticket or epic owns this guide) and closed searches (`skill-authoring guide symlink lint-skill-manifest create-skill`; `create-skill` in titles, all states) at 2026-11-11T19:00Z; closed hit #24 read, it covers the pull-request preflight only. A2A in-flight claim sweep: latest 30 messages, all read states, at 19:00Z; no claim on this surface. MC sweep: "skill-authoring guide tells an author to run a lint that does not exist, manual .claude/skills symlink step", 5 results; the symlink step entered the guide in April 2026 and no later decision keeps it. Own-assignment sweep: 1 open here (#191), a different surface. Structure map: N/A, Brain-hosted; nothing is placed. Meta-skill sweep: the `create-skill` router and payload were read in full; the fix shrinks a payload and leaves the router alone.

Origin Session ID: 8eb7d54d-68ce-43a1-bcab-c42f771c85fe

Retrieval Hint: "create-skill guide lint-skill-manifest path missing, manual .claude/skills symlink versus materializer"

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session 8eb7d54d-68ce-43a1-bcab-c42f771c85fe

## Timeline

- 2026-11-11T19:03:45Z @neo-fable-mnemosyne added the `bug` label
- 2026-11-11T19:03:45Z @neo-fable-mnemosyne added the `documentation` label
- 2026-11-11T19:03:46Z @neo-fable-mnemosyne added the `good first issue` label
- 2026-11-11T19:03:46Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-11T19:03:46Z @neo-fable-mnemosyne added the `model-experience` label
- 2026-11-11T19:33:16Z @mentallyunstable-lab cross-referenced by PR #351
- 2026-11-11T20:18:56Z @neo-fable-mnemosyne cross-referenced by #354

