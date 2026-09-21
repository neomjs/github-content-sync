---
id: 38
title: 'The baseline installs its guards from the workflow''s own commit, not an npm pin'
state: OPEN
labels:
  - enhancement
  - ai
  - build
  - agent-os
assignees: []
createdAt: '2026-09-01T22:46:30Z'
updatedAt: '2026-09-01T22:46:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/38'
author: neo-fable
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
# The baseline installs its guards from the workflow's own commit, not an npm pin

## Context

Operator, 2026-09-01 after merging PR #35: *"hardcoded workflow version numbers make frequent npm publishes a pain."* Tonight showed the cost twice: `#27` filed the pin naming a version that was never published, and the `v0.1.3` cut (`27ad8959`) moved `package.json` without moving the three `SKILLS_VERSION` pins, so the contract test reddened `dev` for 79 minutes until PR #35. `#27`'s own body predicted it: *"nothing mechanically couples 'the pin matches package.json' to 'that version is published' … if this recurs after the publish, that gap — not this instance — is the ticket to file."* It recurred. This is that ticket.

Live latest-open sweep: the 12 open issues in this repository at 2026-09-01T22:40Z (none on pins, versions or publishing), the last 30 A2A messages (all read-states), and the Knowledge Base ticket corpus; no equivalent. PR #36 and #37 (Grace) are unrelated.

## The Problem

`reusable-pr-baseline.yml` installs this package from npm at an exact version in three jobs (`:99`, `:138`, `:183`), and `scripts/test-reusable-pr-baseline.mjs` asserts each pin equals `package.json`'s version. The design intent is right — the guard must not float, or a publish would change every consumer's governance without a diff — but it binds three things that move at different times: the version bump commit, the npm publish, and the workflow file. Every release therefore needs a workflow edit plus a publish, in that order, and the failure modes are both silent until CI: a pin that names an unpublished version (`#27`), or a bump that leaves the pins behind (tonight). The pin is also redundant information: a caller already pins the reusable workflow itself by ref (`uses: neomjs/neo-agent-skills/.github/workflows/reusable-pr-baseline.yml@<ref>`), so the governance version exists in one place already — the caller's ref — and the npm pin is a second copy of it that has to be kept in step by hand.

## The Architectural Reality

- Every job that installs the guard runs `npm install --prefix "${SKILLS_ROOT}" --ignore-scripts --package-lock=false --no-save "neo-agent-skills@${SKILLS_VERSION}"` with `SKILLS_VERSION` a literal in the job's `env`.
- `test-reusable-pr-baseline.mjs` guards the pins with `package version drift` / `substrate package version drift` / `PR-body package version drift` assertions and mutation fixtures that interpolate `pkg.version` (PR #35 fixed the one that did not).
- GitHub Actions exposes, inside a called reusable workflow, `job.workflow_sha` — *"The commit SHA of the workflow file that defines the current job"* (contexts reference) — i.e. the reusable workflow's own commit, together with `job.workflow_ref`. That is the immutable revision the caller pinned, available to the job without any input.
- `package.json` `files` whitelists `.agents/skills`, the four `bin` scripts and the README, and `repository.url` is the GitHub repo, so the package is installable from a checkout or a git URL with `--ignore-scripts` exactly as it is from the registry.
- The README's promise for both guard jobs is *"installs its exact guard release outside the caller workspace, so a pull request cannot weaken its own gate"* — the invariant to keep is "exact and outside the caller's tree", not "from npm".

## The Fix

1. **Install the guard from the workflow's own commit.** In each guard job, check out this repository at `${{ job.workflow_sha }}` into `${{ runner.temp }}` (`actions/checkout` with `repository: neomjs/neo-agent-skills`, `ref`, `path`) and `npm install --prefix "${SKILLS_ROOT}" --ignore-scripts --package-lock=false --no-save "<that path>"`. The guard code is then, by construction, the exact commit of the workflow file the caller pinned: one governance revision, chosen once by the caller's `@<ref>`, with no version literal anywhere in the workflow. A variant with the same property is a git-URL spec (`git+https://github.com/neomjs/neo-agent-skills.git#${{ job.workflow_sha }}`); the checkout form is preferred because it needs no git-over-npm semantics on the runner.
2. **Retire the pins and their drift assertions**; replace them with contract assertions that each guard install references `job.workflow_sha` and a `runner.temp` path, and mutation fixtures that turn either into a literal / `latest` / a caller-workspace path. The "exact, outside the caller's tree" invariant keeps its witnesses; the "equals `package.json`" invariant disappears because nothing needs it.
3. **npm publishes become independent of the baseline.** The registry remains the distribution path for consumers that run `npx neo-agent-skills-…` directly; the reusable workflow no longer waits for, or breaks on, a publish.
4. **Fallback if the checkout form is refused in practice** (e.g. a policy against repo checkouts in guard jobs): keep an exact npm pin but declare it ONCE at the workflow level (`env.SKILLS_VERSION`) and have the `version` lifecycle script rewrite it so `npm version <bump>` moves `package.json` and the pin in one commit. That removes the manual edit and the bump/pin race, but keeps the publish ordering; it is the lesser fix.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| guard install in `reusable-pr-baseline.yml` (3 jobs) | `job.workflow_sha` (Actions contexts reference) + README "exact guard release outside the caller workspace" | installs this repository at the reusable workflow's own commit into `runner.temp` | none — a missing checkout fails the job at install, before any governance runs | workflow comments + README §baseline | contract test + one caller run |
| `scripts/test-reusable-pr-baseline.mjs` pin assertions | this ticket | assert the install references `job.workflow_sha` and a `runner.temp` path; mutations to a literal, `latest`, or the caller workspace go red | n/a | inline | the suite's own mutation count |
| npm publish ↔ baseline coupling | this ticket | none — a publish never edits the workflow and the workflow never waits for a publish | n/a | README release section | `git grep SKILLS_VERSION` → 0 hits |

**Decision Record impact:** none.

## Acceptance Criteria

- [ ] No `SKILLS_VERSION` literal and no `package.json` version assertion remain in `reusable-pr-baseline.yml` or its contract test; each guard job installs from a checkout of this repository at `${{ job.workflow_sha }}` under `runner.temp`.
- [ ] The contract test asserts the new shape and goes red on a version literal, a `latest` spec, or an install from the caller's workspace (mutation fixtures, interpolation-free by construction).
- [ ] One caller run proves the mechanism end to end: a consumer workflow calling the baseline at a pinned ref reaches every guard job past its install step, and the installed guard's `package.json` version equals this repository's version at that commit (log receipt in the PR).
- [ ] A version bump commit that touches only `package.json` leaves the contract test green — the bump/pin race cannot recur.
- [ ] README's baseline section says where the guard revision comes from (the caller's ref) and that publishing is independent of it.

## Out of Scope

- The publish cadence or the release pipeline itself (#22, #23).
- Consumer-side `npx neo-agent-skills-…` calls, which keep resolving from the registry.
- Wiring the `neomjs/neo` caller (neomjs/neo#17783).

## Avoided Traps

- **Floating the pin** (`latest`, `^0.1`) — `#27`'s reasoning stands: a publish must never change a consumer's governance without a diff. The workflow commit is exact by construction.
- **Moving the pin to the callers** (a `skills_version` input) — N copies instead of three.
- **Version-bump automation alone** (the fallback) — it fixes the race, not the coupling; a publish is still a precondition for the baseline.

## Related

#27 (the pin names an unpublished version) · PR #35 (tonight's instance fix) · #14 (governance unification) · neomjs/neo#17783 (the engine caller) · neomjs/neo#17175 (the byte budget the substrate guard exists for)

Origin Session ID: 6197a12f-acf2-478a-b05d-4ece32474259

Retrieval Hint: "reusable baseline job.workflow_sha install guard from own commit no npm pin SKILLS_VERSION retired"

🪢 Mnemosyne (Claude Fable 5.1 · Claude Code) · session 6197a12f-acf2-478a-b05d-4ece32474259

## Timeline

- 2026-09-01T22:46:31Z @neo-fable added the `enhancement` label
- 2026-09-01T22:46:32Z @neo-fable added the `ai` label
- 2026-09-01T22:46:32Z @neo-fable added the `build` label
- 2026-09-01T22:46:32Z @neo-fable added the `agent-os` label
- 2026-09-08T15:25:19Z @neo-opus-grace cross-referenced by PR #18485
- 2026-09-16T08:58:36Z @neo-opus-vega cross-referenced by #80
- 2026-09-16T10:40:05Z @neo-opus-vega cross-referenced by PR #82
- 2026-09-18T12:19:32Z @neo-opus-vega cross-referenced by #90

