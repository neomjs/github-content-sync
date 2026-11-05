---
id: 276
title: 'Seats never reinstall the skills they load: 9 of 10 run weeks behind npm'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-11-05T12:12:54Z'
updatedAt: '2026-11-05T12:21:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/276'
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
closedAt: '2026-11-05T12:21:04Z'
---
# Seats never reinstall the skills they load: 9 of 10 run weeks behind npm

## Context

The operator observed on 2026-11-05 that two Claude seats (Vega, Grace) self-cleared sessions often while three (Ada, Clio, Mnemosyne) did not. His hypothesis was that the three never ran `npm i`, so their engine clones loaded old skills. I measured each seat's `neomjs/neo/node_modules/neo-agent-skills/package.json` and its installed `session-sunset` text at 12:08Z:

| Seat | Installed (date) | `clear_session` clause (#145) | 250k gate (#251, ≥ 0.1.79) |
|---|---|---|---|
| Sophie | 0.1.29 (10-04) | no | no |
| Clio | 0.1.39 (10-11) | yes | no |
| Julia | 0.1.41 (10-14) | yes | no |
| Maryam | 0.1.41 (10-17) | yes | no |
| Ada | 0.1.73 (10-30) | yes | no |
| Mnemosyne | 0.1.78 (10-30) | yes | no |
| Vega | 0.1.78 (11-02) | yes | no |
| Emmy | 0.1.78 (11-02) | yes | no |
| Euclid | 0.1.82 (11-03) | yes | yes |
| Grace | 0.1.84 (11-03) | yes | yes |

npm latest is 0.1.87 (0.1.88, with #275, is publishing), and neo pins 0.1.86. Nine of ten seats are behind npm, two by more than 45 versions. The hypothesis holds for Ada, Clio and Mnemosyne: they never loaded #251's gate. Vega wrote that gate, so it was in her context without the install. I had it installed.

## The Problem

A skills change reaches a seat only when that seat runs `npm install` in its neo clone after pulling. `.claude/skills/*` symlinks into `node_modules/neo-agent-skills` (neo's `dependabot.yml` comment), and other harnesses materialize from the same package. Nothing runs that install:
- `session-sunset` Step 1 pulls `dev` and reconciles the Brain config, with 0 install commands.
- I found no install step in the Brain's lifecycle scripts or the Institution's sources by text grep. A spawn-array call could evade that grep, but the install dates above are the receipt either way.

So the friction→gold loop stops at `node_modules`. #140 named this risk for 0.1.19 ("every seat keeps loading 0.1.19"), and its load-receipt AC has been open since 2026-10-04. #275 reaches a seat only when that seat reinstalls.

**A trap for the catch-up:** reinstalling at neo's current pin (0.1.86) installs #251's gate. The reinstall wave has to follow neo's bump to ≥ 0.1.88.

## The Architectural Reality

- `references/session-sunset-workflow.md` Step 1 (primary clone): `git switch dev`, `git pull origin dev`, then `--migrate-config` where the Brain script exists. It has no install.
- `references/context-recovery-workflow.md` §3 is the boot sequence every fresh session runs, and it has no installed-vs-pinned check.
- `npm install` runs `prepare` (`buildScripts/util/prepare.mjs`), which re-materializes the harness skill links.
- Design authority: #140, *"Validation must run against the text a session actually loads, from the recipient's own session"*. The intended state is that a session loads the pinned package.
- Decision Record impact: none. Structure map: N/A, since two skill payloads change and no `.mjs` is added.

## The Fix

1. **`session-sunset` Step 1:** after the pull, in a clone whose `package.json` pins `neo-agent-skills`, run `npm install`. It is best-effort like the pull, and a failure goes into the handover.
2. **`context-recovery` §3:** before the first skill read, compare the installed `node_modules/neo-agent-skills` version with the clone's pin. On drift, pull and reinstall first. This covers seats that rarely sunset, the very ones the table shows behind.
3. **One-time catch-up**, after neo pins ≥ 0.1.88: every seat pulls and reinstalls. A broadcast names the version to verify.
4. Version bump; net loaded bytes flat or lower.

## Acceptance Criteria

- [ ] Sunset Step 1 reinstalls after the pull, best-effort and surfaced in the handover.
- [ ] Context-recovery §3 checks installed against pinned skills before any skill read, and reinstalls on drift.
- [ ] After neo pins ≥ 0.1.88, a census of all ten seats (the table above, re-run) shows each one at the pin.
- [ ] `npm run lint -- --base origin/dev` is green, the version is bumped, and net bytes are flat or lower.

## Out of Scope

- A Fleet Manager install on seat restart (Institution). It would be a stronger guarantee; it gets its own ticket if these two steps prove insufficient.
- #140's D#19384 load receipts (Clio's), which this unblocks.

## Related

#140 · #274 / #275 · #251 · #145

Live latest-open + closed sweep: I checked the latest 20 open issues, plus a closed search for `npm install`, `node_modules`, `reinstall`, `skills version` and `stale skills`, and an org-wide open search, at 2026-11-05T12:11Z. #140 is the closest; it is a one-change integration close, not the mechanism. No equivalent ticket exists.

Origin Session ID: 0ec8e189-2090-4833-9e9c-4376ea22f341
Retrieval Hint: "seats never reinstall neo-agent-skills node_modules stale versions sunset step 1 npm install drift check"

## Timeline

- 2026-11-05T12:12:56Z @neo-opus-grace added the `bug` label
- 2026-11-05T12:12:56Z @neo-opus-grace added the `ai` label
- 2026-11-05T12:12:56Z @neo-opus-grace added the `agent-os` label
- 2026-11-05T12:12:58Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-05T12:20:55Z @neo-opus-grace cross-referenced by #1480
### @neo-opus-grace - 2026-11-05T12:21:01Z

**Closing as superseded by neomjs/neo-agent-brain#1480** (I am the author). This ticket prescribed two more skill steps: a sunset reinstall and a recovery drift check. The census here is itself the argument against that shape: the reinstall rule already existed as discipline, and nine of ten seats still drifted. The operator's direction, *"automation beats discipline … peers would not even need to think about it then"*, moves the fix into Fleet's Start path. Before every launch, a reused checkout gets a fast-forward and an `npm install`. The census table and the 0.1.88 sequencing trap carry over to #1480, and so does the deletion it names: `session-sunset` Step 1's pull retires once Start does the refresh.

Origin Session ID: 0ec8e189-2090-4833-9e9c-4376ea22f341

- 2026-11-05T12:21:04Z @neo-opus-grace closed this issue

