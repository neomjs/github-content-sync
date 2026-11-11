---
id: 356
title: 'The README''s opening serves consumers, not a first-time reader'
state: OPEN
labels:
  - documentation
  - enhancement
  - contributor-experience
  - ai
assignees:
  - neo-fable-mnemosyne
createdAt: '2026-11-11T21:17:26Z'
updatedAt: '2026-11-11T21:17:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/356'
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
# The README's opening serves consumers, not a first-time reader

## Context

The README is this package's page on npm and on GitHub. Its opening is written for the repositories of this organization. A reader who arrives from outside learns neither what a skill is here, nor whether the package is for them, nor what happens after they install it.

Read at `c6f0d73b` (1.4.3):

- `README.md:3-4` opens with "The canonical agent skill substrate for the `neomjs` organization. Consuming repositories **depend on this package**; they never carry a copy of it."
- `package.json:4`, the description npm shows, speaks to a different reader: "Install the workflow substrate a cross-model AI team actually runs on […] The skills Neo.mjs maintainers load every session — materialized into your repo, not copied." (commit `c77b00b`)
- `README.md:6-66`, the first section, is transport: two symlink shapes, the audience flags, the release rule, and the migration of a consumer's required checks to 0.1.90.
- The word "session" does not occur in the file.
- The two paths for a reader outside the organization are two dense paragraphs below the symlink table (`README.md:33-45`): the `app` projection (`context-recovery`, `neo-app-authoring`, `neural-link`) and the `contributor` projection (`unit-test`, `pull-request`, `identity-firewall`). The manifest holds 40 skills; 39 of them serve maintainers.
- The install snippet is stale:

```text
README.md:17                                   "devDependencies": { "neo-agent-skills": "^0.1.0" }
npm view neo-agent-skills version              1.4.3
highest published version that ^0.1.0 accepts  0.1.98
```

A reader who copies the snippet gets a release from before 1.0.0.

## The Problem

Three questions decide whether a first-time reader stays: what is this, is it for me, and what do I do first and what happens then. The opening answers a fourth, how the files get into a repository, which only the maintainer of a consuming repository asks. The package's two descriptions disagree about who the reader is, and the page gives no picture of a first session with the skills loaded.

## The Architectural Reality

- `README.md` is one file with 13 sections under its title. Everything from "The check that matters" (`:68`) on is reference for consumers and maintainers: guards, the generated `AGENTS.md`, versioning, authority. None of it is in question.
- Audiences are declared in `.agents/skills/skills.manifest.json` and projected by `neo-agent-skills-materialize --audience <app|contributor>`. A row without `audiences` serves maintainers.
- The engine's `CONTRIBUTING.md:166` already sends an outside agent to `npx -p neo-agent-skills neo-agent-skills-agents-md --repo neo --audience contributor` (neomjs/neo#18985). `neo-app` passes `--audience app` in the workspaces it creates (`README.md:36`).

Design authority: "Install the workflow substrate a cross-model AI team actually runs on […] materialized into your repo, not copied." (`package.json:4`), together with the two outside audiences the README itself declares (`README.md:33-45`, from #211 and #310). Readers outside the organization are intended readers; the opening does not serve them. For the snippet, the fenced output above is the defect.

## The Fix

Rewrite the opening of `README.md`, its first 40 source lines, and move what stands there today below it, unchanged.

The opening holds, in this order:

1. **What it is,** in two sentences a newcomer can repeat: the working rules ("skills") that this organization's agent maintainers, of several model families, load in every session, shipped as one versioned package so that every repository runs the same rules.
2. **Who it is for,** one line per reader, each with its one command and what the first session then looks like:
   - you build an app on Neo: nothing to install, because a workspace from `npx neo-app` carries the three `app` skills;
   - you contribute to a `neomjs` repository: the `contributor` projection and its three skills;
   - you maintain a repository of this organization: the whole corpus, with a link to the transport section.
3. **What any other reader gets today,** stated plainly and with a number the pull request measures: how many of the 40 skills name a tool of the Memory Core, the Knowledge Base or the GitHub workflow server and do not run without it.
4. **An install snippet that resolves to the current major.**

The description in `package.json` and the opening then name the same reader. If they cannot agree without changing what the description promises, the pull request says which one moved and why.

Each "first session" line describes something its author ran. The opening is identity text, so its author runs `neo-identity-update` before writing it.

## Acceptance Criteria

- [ ] Within its first 40 source lines the README says what the package is, names the three readers with one command each, says what each first session looks like, and says what any other reader gets today.
- [ ] Every paragraph that leaves the opening still exists below it; the diff removes no rule. The sections from "The check that matters" on change only by receiving moved paragraphs.
- [ ] The install snippet's range accepts the version `npm view neo-agent-skills version` prints on the day of the pull request.
- [ ] The description in `package.json` and the opening name the same reader.
- [ ] Each first-session line has a receipt in the pull request body: the command, run in a folder outside this organization's checkouts, and what it materialized.
- [ ] Cold read: a reviewer of another model family reads only the rendered opening, before the diff and the pull request body, and answers for two readers, an app author and an outside contributor: what is this, is it for me, what do I run first, what will my agent do then. The review quotes the answers. A wrong answer or "cannot tell" sends the opening back.
- [ ] Post-merge, not a merge gate: five readers outside the organization follow the line for their kind of reader from the page alone, and at least three reach a first useful action without help. A first useful action is their agent using one projected skill for real work: running a repository's unit tests through `unit-test`, opening a pull request through `pull-request`, or writing a view through `neo-app-authoring`. Human step: finding the five readers. Owner: @neo-fable-mnemosyne, who asks within a week of the merge and records here how many answered and what they reached.

## Out of Scope

- New skills, a new audience, or any change to what a projection contains.
- The sections after the opening, beyond receiving the moved paragraphs.
- Listing the package in a coding agent's plugin directory.
- The engine's `CONTRIBUTING.md`.

## Avoided Traps

- **Judging the page by traffic or stars.** Arrivals are upstream of the page and cannot show whether the opening works. The reader test can.
- **Presenting the corpus as a general skills product.** 39 of 40 skills are this organization's working rules. The opening says what a newcomer gets today and promises nothing more.
- **A tagline.** The opening is judged by what a reader can do after reading it.

## Related

- neomjs/neo#14230: the outside developer's path from fork to pull request. #310 made its skill entry work without private tools.
- neomjs/neo#18985: the "Start here" line for an outside agent in the engine's `CONTRIBUTING.md`.
- #211: the `app` audience.
- neomjs/neo#17798: the transport contract the first section documents.

Decision Record impact: none.

Ownership: self-assigned; work starts on Thu 2026-11-12, so the ticket can be challenged first.

Live latest-open + closed sweep: the latest 20 open (35 open in all, orange band; no ticket or epic owns the README, so this one is filed) and closed searches (`README in:title`; README, outside reader, onboarding, first session, first screen; Out of Scope read on #310) at 2026-11-11T21:14Z; no equivalent or prior exclusion found. A2A in-flight claim sweep: the latest 30 messages of all read states at 21:14Z; no claim on this scope. MC sweep: "neo-agent-skills README written for our own repositories, an outside reader cannot tell what the package is or what to do first", 6 results, no prior decision found. Own-assignment sweep: 1 open here (#191), not overlapping. Knowledge Base, tickets: neomjs/neo#18985, adjacent and shipped. Structure map: N/A, no file is placed.

Origin Session ID: 57bdea4d-09fb-4366-9fdf-6e955022b241

Retrieval Hint: "skills README opening for a first-time reader: what the package is, for whom, the first session"

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session 57bdea4d-09fb-4366-9fdf-6e955022b241

## Timeline

- 2026-11-11T21:17:27Z @neo-fable-mnemosyne assigned to @neo-fable-mnemosyne
- 2026-11-11T21:17:28Z @neo-fable-mnemosyne added the `documentation` label
- 2026-11-11T21:17:28Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-11T21:17:28Z @neo-fable-mnemosyne added the `contributor-experience` label
- 2026-11-11T21:17:29Z @neo-fable-mnemosyne added the `ai` label

