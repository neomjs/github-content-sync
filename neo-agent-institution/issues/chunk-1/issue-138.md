---
id: 138
title: 'Both Skills pins sit at 0.1.3, and CI reads them in two different jobs'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - dependencies
assignees:
  - neo-opus-grace
createdAt: '2026-09-15T13:33:15Z'
updatedAt: '2026-09-15T13:44:43Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/138'
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
closedAt: '2026-09-15T13:44:43Z'
---
# Both Skills pins sit at 0.1.3, and CI reads them in two different jobs

## Context

`.github/workflows/shared-pr-baseline.yml` calls the Skills reusable workflow at an exact commit — `reusable-pr-baseline.yml@7c9bd16f`, the Skills `dev` head at adoption on 2026-09-04. The workflow's own header says this is deliberate and that moving it is a reviewed act:

> An exact commit, never a branch or tag: the coordinate is the Skills `dev` head at adoption (2026-09-04) … Moving it is a reviewed change here, one line.

That pin is now two releases behind. `7c9bd16f` hardcodes `SKILLS_VERSION: '0.1.3'`, so **every job this repository runs through the shared baseline installs the 0.1.3 guards** — source archaeology, substrate size, and the agent PR-body gate.

`neomjs/neo` hit the same drift today from the other direction and it is what surfaced this: its dependency bump to 0.1.5 left CI still installing 0.1.4, because the dependency and the workflow pin are **different coordinates**. The archaeology job then flagged the exact colour literals 0.1.5 exists to pass. Fixed there in [neomjs/neo#18734](https://github.com/neomjs/neo/pull/18734) by moving the pin to `27e151d9`.

Live latest-open sweep: latest 10 open issues read 2026-09-15T13:2xZ; no equivalent. Full-text sweep over open and closed for `skills version bump baseline pin`: nothing. A2A claim sweep over the last 30 messages, all read-states: no competing claim — @neo-opus-ada is on `#18731`, @neo-opus-vega on `#15202`.

## The Problem

Between 0.1.3 and 0.1.5 the archaeology guard learned three rules, all of which **narrow** what it reports:

- a numeric CSS colour of 3/4/6/8 digits directly after colour syntax (`color:`, `backgroundColor_=`, `fillStyle=`, `CSS color`) is a colour and needs no marker;
- a number with a leading zero is never a ticket;
- a numeric HTML entity (`&#39;`, `&#8212;`) is a codepoint, never a ticket.

So on this repository today, a durable comment containing `backgroundColor_='#000000'` or `renders &#8212;` fails the shared baseline while the identical comment passes in `neomjs/neo`. **Two consumers of one governance substrate disagree about the same line** — which is the exact defect `neomjs/neo#16553` was opened to end, still live here.

### Two stale pins, and CI reads both — in different jobs

**Correction to this ticket's first revision.** It claimed the `package.json` range "already resolves to 0.1.5" and that the workflow SHA was "the only stale coordinate." Measured, both halves are wrong:

```
package.json            "neo-agent-skills": "^0.1.3"
package-lock.json       node_modules/neo-agent-skills -> 0.1.3
node_modules (installed)                                 0.1.3
npm registry latest                                      0.1.5
```

A range **permits** a version; the lockfile **pins** one. Nothing here resolves forward on its own.

And the two pins are not redundant, because two different jobs read them:

| job | how it resolves the guard | pin it reads |
|---|---|---|
| `skills-materialized` | `npx --no-install neo-agent-skills-materialize --check` — the **caller's** `node_modules` | the **lockfile** |
| `source-comment-archaeology`, `substrate-size`, `pr-body` | `npm install --prefix "$RUNNER_TEMP" neo-agent-skills@${SKILLS_VERSION}` | the **`uses:` SHA** |

Moving only the SHA would have left `skills-materialized` checking against the 0.1.3 materializer. Both coordinates move, or the repository stays split against itself.

## The Architectural Reality

- `.github/workflows/shared-pr-baseline.yml` — the caller; owns only the event trigger and the immutable Skills coordinate.
- `neomjs/neo-agent-skills@27e151d9` — the merge commit of [neo-agent-skills#68](https://github.com/neomjs/neo-agent-skills/pull/68), carrying `SKILLS_VERSION: '0.1.5'` at all three call sites (archaeology, substrate size, PR body).
- The immutability is the point: a consumer PR cannot weaken the gate it is judged by. The cost is that a guard release reaches a consumer only through a reviewed pin move — this ticket.

## The Fix

Move the `uses:` coordinate in `shared-pr-baseline.yml` from `@7c9bd16f97a9b58309602431fc5b5741bf818e22` to `@27e151d9c427e07c674fcccaff1382d2a33d0b24`, update the adoption note in the surrounding comment so the recorded coordinate and rationale match, and move the dependency floor to `^0.1.5` with the lockfile refreshed to match.

## Acceptance Criteria

- [ ] **AC-1** — `shared-pr-baseline.yml` pins `reusable-pr-baseline.yml@27e151d9c427e07c674fcccaff1382d2a33d0b24`.
- [ ] **AC-2** — that coordinate is confirmed to carry `SKILLS_VERSION: '0.1.5'` at every call site, read from the Skills repository at that SHA rather than assumed.
- [ ] **AC-3** — `package-lock.json` resolves `neo-agent-skills` to 0.1.5, so `skills-materialized` and the isolated-install jobs agree on one release.
- [ ] **AC-4** — the shared baseline is green on this PR, which is itself the evidence that the newer guards accept this repository's current tree.
- [ ] **AC-5** — the workflow's adoption comment names the new coordinate and date; a stale rationale beside a moved pin is how the next reader mis-dates the contract.

## Out of Scope

- **`neomjs/devindex`**, which pins `reusable-pr-baseline.yml@72965ba5` (2026-08-29) — a coordinate predating `SKILLS_VERSION` entirely, so it carries no version key. That is a different shape and needs its own read; noted here so the sweep is not repeated.
- Any change to the guards themselves, or to which jobs the baseline runs. The job set is byte-identical between `7c9bd16f` and `27e151d9`; 0.1.5 ships two further binaries (`agents-md`, `workflow-concurrency`) that the reusable workflow does not invoke.

## Avoided Traps

- **Do not replace the SHA with a branch or tag.** The immutability is the mechanism, not an inconvenience — a consumer PR must not be able to move the gate it is judged by.
- **Do not read a permissive range as a current install.** `^0.1.3` accepts 0.1.5 and had installed 0.1.3 for eleven days. The falsifier is `node -p "require('./node_modules/neo-agent-skills/package.json').version"`, not the range.
- **Do not read this PR's own archaeology green as coverage.** Its diff contains no auditable source file, so the guard reports zero files and greens vacuously. The positive control is a base whose diff carries `.mjs`.

## Decision Record impact

none.

## Related

`neomjs/neo#16553` (the rule the two guards must agree on) · [neomjs/neo#18734](https://github.com/neomjs/neo/pull/18734) (the same pin move in the engine) · [neo-agent-skills#68](https://github.com/neomjs/neo-agent-skills/pull/68) (the 0.1.5 rules) · `neo-agent-skills#47`

Origin Session ID: 148d12cc-9777-46e4-bfb9-a422479153a5

Retrieval Hint: `query_raw_memories("reusable-pr-baseline SKILLS_VERSION pin consumer drift 0.1.3 0.1.5")`


## Timeline

- 2026-09-15T13:33:15Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-15T13:33:17Z @neo-opus-grace added the `enhancement` label
- 2026-09-15T13:33:17Z @neo-opus-grace added the `agent-os` label
- 2026-09-15T13:33:17Z @neo-opus-grace added the `ai` label
- 2026-09-15T13:33:18Z @neo-opus-grace added the `dependencies` label
- 2026-09-15T13:39:39Z @neo-opus-grace changed title from **The shared PR baseline pins Skills at 0.1.3, two guard releases behind** to **Both Skills pins sit at 0.1.3, and CI reads them in two different jobs**
- 2026-09-15T13:41:05Z @neo-opus-grace cross-referenced by PR #139
- 2026-09-15T13:44:43Z @tobiu referenced in commit `5eeb43d` - "Merge pull request #139 from neomjs/grace/138-skills-015

chore(ci): both Skills pins move to 0.1.5, and CI reads them in different jobs (#138)"
- 2026-09-15T13:44:43Z @tobiu closed this issue
- 2026-09-15T16:42:56Z @neo-opus-vega cross-referenced by PR #143

