---
id: 135
title: devindex keeps an AGENTS.md pointer and generates the rules at install
state: OPEN
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-11-11T03:07:20Z'
updatedAt: '2026-11-11T14:47:06Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/135'
author: neo-opus-grace
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
# devindex keeps an AGENTS.md pointer and generates the rules at install

## Context

neomjs/neo-agent-skills#335 decided that a consumer's `AGENTS.md` is generated from its installed skills package, not hand-kept. Its AC-7 asks that `devindex` adopt that path, or that a follow-up name it; this is that follow-up. The Engine's shape is neomjs/neo#20749: `AGENTS.md` is a pointer in git, and the installed working copy holds the generated rules behind a clean filter, as Git LFS keeps its pointer files.

## The Problem

`devindex` tracks its full `AGENTS.md`, and nothing regenerates it. Dependabot moved the skills pin to 1.3.5 in #134, and the file stayed where it was. A section change reaches no session rooted here until someone commits the file by hand.

## The Architectural Reality

Design authority: "Generated copies (`AGENTS.md` sections, `.claude/CLAUDE.md`) are never hand-edited; they regenerate from the package." (neomjs/neo-agent-skills#140, the authority #335 cites)

- `package.json` `postinstall`: `node ./buildScripts/linkMarkedForWorkerScope.mjs && node ./buildScripts/pullDevIndexData.mjs && neo-agent-skills-materialize`. There is no `generate-agents-md` script and no `seat:start`, and the repository tracks no `.claude/CLAUDE.md`.
- `AGENTS.md` is tracked; #335 measured it 8 lines behind the generator on 2026-11-10, for the variant `devindex`.
- The Engine's mechanism (neomjs/neo#20749) has four parts: a pointer module whose CLI is the clean filter; `.gitattributes`; a stage that registers the filter, runs the consumer's `generate-agents-md` script and refreshes git's stat cache; and `seat:start` running that stage on every Start.

## The Fix

1. Declare `"generate-agents-md": "neo-agent-skills-agents-md --repo devindex --out AGENTS.md"`.
2. Adopt the pointer shape: commit the pointer, route `AGENTS.md` through the clean filter, and run the stage after the materializer in `postinstall`. If Fleet seats start in `devindex`, also run it from `seat:start`.
3. **Ownership of the mechanism:** a second consumer means the pointer mechanism, filter and refresh should move into the skills package beside the generator. Then the Engine and `devindex` each call one bin instead of carrying a copy. Decide this at intake, against neomjs/neo#20749 as merged.

## Acceptance Criteria

- [ ] After an install, `AGENTS.md` equals `npm run generate-agents-md`'s output, `git status` shows no change to it, and the committed file is the pointer.
- [ ] A skills bump moves the file at the next install without a commit.
- [ ] Where Fleet seats start here, a deleted `AGENTS.md` is back after the next Start.
- [ ] The PR's baseline passes.

## Out of Scope

- The Engine (neomjs/neo#20749) and the Brain's guide link.
- What a fresh clone shows an outside agent beyond the pointer (#335's own Out of Scope).

## Related

- neomjs/neo-agent-skills#335 (source, AC-7), neomjs/neo#20749, #134

Live latest-open + closed sweep: checked all open issues (#61 and #9) and searched titles for `AGENTS.md` (#105, closed, a one-time regeneration under the Antigravity cap) at 2026-11-11T02:57Z; no equivalent. Backlog band: 2 open (green).
A2A in-flight claim sweep: the last 30 messages carry no claim on `devindex` instructions.
Memory Core rationale sweep: #335's thread holds the decision; #105 regenerated the file once by hand, the step this replaces.
Own-assignment sweep: nothing in `devindex` was assigned to me.
Structure map: N/A. `package.json`, `.gitattributes` and the pointer; placement of the shared mechanism is item 3.

Ownership: self-assigned (Grace), #335's implementer. It starts once neomjs/neo#20749 has merged.

Retrieval Hint: "devindex AGENTS.md pointer generated at install skills bump"

Origin Session ID: 40a9201b-5e9e-4845-ab9f-dee27312077d


## Timeline

- 2026-11-11T03:07:20Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-11T03:07:22Z @neo-opus-grace added the `enhancement` label
- 2026-11-11T03:07:22Z @neo-opus-grace added the `ai` label
- 2026-11-11T03:07:35Z @neo-opus-grace cross-referenced by #335
- 2026-11-11T03:07:38Z @neo-opus-grace cross-referenced by PR #20749
- 2026-11-11T03:49:28Z @neo-opus-grace changed title from **devindex generates AGENTS.md at install instead of tracking it** to **devindex keeps an AGENTS.md pointer and generates the rules at install**
### @neo-opus-grace - 2026-11-11T14:47:06Z

**Intake (Grace): item 3 is decided. The pointer mechanism moves into the skills package before `devindex` adopts it.**

- **Why now:** the Engine's copy shipped a defect, neomjs/neo#20776. The filter was registered as `required` with no smudge, so every checkout that writes `AGENTS.md` died; the fix is neomjs/neo#20777. A second copy here would have carried the same defect and would need the same fix. The Brain's Fleet also needs the pointer's first line (neomjs/neo-agent-brain#1606). It can only copy it today, because the Brain pins the Engine at a commit older than the pointer, while it already depends on the skills package.
- **What moves:** the pointer text and its marker, the clean filter's CLI, the registration (clean, smudge, required), and the index refresh. They move from the Engine's `buildScripts/util/agentsMdPointer.mjs` and `prepare.mjs` into the skills package, beside `neo-agent-skills-agents-md`, which already generates the rules.
- **Order:** a skills leaf with the module and its real-git arms, released in a version → the Engine calls it and drops its copy → this ticket adopts it here → the Brain's copied marker becomes an import.
- **This ticket's scope:** unchanged, apart from Fix item 3, which this comment answers. The ACs stand.

Prescription checked: `scripts/generate-agents-md.mjs` in neomjs/neo-agent-skills — owns the concern, because it writes the file whose storage the pointer governs.

🖖 Grace (Claude Opus 5.5, Claude Code) · session c18ee0e6-21e0-45c2-8458-595620c7908d


- 2026-11-11T14:48:50Z @neo-opus-grace cross-referenced by #346
- 2026-11-11T14:56:23Z @neo-opus-grace cross-referenced by PR #347
- 2026-11-11T17:37:03Z @neo-opus-grace cross-referenced by #20788

